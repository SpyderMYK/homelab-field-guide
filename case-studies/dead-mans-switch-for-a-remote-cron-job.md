# Case study: a dead-man's switch for a remote scheduled job

**Goal:** get notified if a twice-daily batch job on a *remote* machine — one you reach only over a
mesh VPN and don't fully control — **stops running.** The hard part of "did this run?" monitoring is
that a healthy silence and a broken pipeline look identical. This is the design journey, including
two approaches I built or evaluated and then rejected, and *why*.

Generic throughout. Cast: `remote` (the machine running the job, reachable only over the mesh VPN)
and `obs` (your observability host: Prometheus + Grafana).

---

## Requirements

1. Alert if the job hasn't succeeded in ~24h (it runs twice daily).
2. **Silence must be meaningful** — a broken alerting path must not look like "all good."
3. Don't leak the job's actual *data* off the remote machine; only health/liveness.
4. Survive the remote machine logging out / sleeping without crying wolf.

---

## Attempt 1 — Prometheus Pushgateway. Built, tested, **rejected**.

The obvious reach: the job pushes a "last success" timestamp to a Pushgateway that Prometheus
scrapes. I wired it up and it worked end-to-end. Then I read the upstream guidance and tore it out.

Prometheus's own docs:

> *The only valid use case for the Pushgateway is capturing the outcome of a **service-level** batch
> job — one **not** semantically related to a specific machine or instance.* For machine-level jobs,
> use the **node_exporter textfile collector** instead.

This job is emphatically machine-level (it's *that* remote box). Two concrete problems confirmed it:

- **No TTL.** Pushgateway is a cache; a pushed metric persists **forever** until manually deleted.
  I hit this within minutes — my test metric had to be `DELETE`d by hand. Stale liveness data is
  worse than none.
- **It decouples the metric's lifetime from the host's**, which is the opposite of what a
  dead-man's switch needs.

**Lesson:** when your design fights the documentation, the docs usually win. Reaching for the
fancier component added failure modes for no benefit.

## Attempt 2 — a hosted heartbeat service. Evaluated, **not chosen**.

Purpose-built dead-man's-switch services (the `healthchecks.io` model) are the *textbook* fit: the
job pings a URL on success; no ping within the grace window → alert. It also solves a flaw the
in-house options don't (below). I did a real trust review — open-source (self-hostable exit path),
2FA, encrypted in transit, sensible jurisdiction — all good. The honest downside was operational: a
single-maintainer service with no SLA, and it would hold the notification credential.

Perfectly reasonable choice for many people. I went in-house for full control — but **it flagged the
one real weakness in the in-house design**, so it earned its place in the writeup.

## Attempt 3 — node_exporter textfile collector. **Chosen.**

The Prometheus-recommended path for machine-level jobs, and it gives back something the others
couldn't: `up{}` (host-down detection) and the remote host's metrics, for free.

**On the remote machine:**

1. Run `node_exporter` with the **textfile collector** enabled, **bound to its mesh-VPN address
   only** — never `0.0.0.0`. node_exporter has no auth and exposes detailed host metrics; on a
   network you don't control, a wide bind is a real exposure.
2. On each successful run, the job writes a `.prom` file the collector picks up:

   ```
   # HELP job_last_success_timestamp_seconds Unix time of last successful run
   # TYPE job_last_success_timestamp_seconds gauge
   job_last_success_timestamp_seconds 1700000000
   ```

   **Write it atomically** — temp file in the *same* directory, then `os.replace()`. node_exporter
   reads whole files; a half-written one throws a scrape error.

**On `obs`:** Prometheus scrapes the remote over the mesh VPN, and Grafana holds the alert.

### The alert query — and why the naive version is wrong

First cut: alert when the metric is too old, with `No Data → Alerting` so a missing series also
fires. That would have **paged every night the remote user logged out** (the exporter only ran while
logged in). Fix: look *back* far enough to find the last success even when the series is currently
absent:

```promql
time() - max_over_time(job_last_success_timestamp_seconds[48h]) > 86400
```

`max_over_time(...[48h])` tolerates logout/sleep gaps while still catching a *genuinely* dead job.
Keep `No Data → Alerting` as the backstop for the truly-gone case, with a short `for:` to debounce.

### Making silence meaningful (requirement #2)

The job also sends, on the same notification path as the alerts:

- **high-priority failure alerts** (a run failed), and
- a **once-daily low-priority heartbeat**.

The heartbeat is the point: if you stop seeing the daily ping, *that's* the signal. Without it, a
dead alerting path is indistinguishable from "nothing wrong" — a trap I've fallen into more than once
(an alerter emailing a local user with no mail transport; a dashboard posting to a topic nobody was
subscribed to). **Test the full chain with a real event and confirm it lands on your actual device**
— an HTTP 200 from the push service only proves it was *accepted*, not *delivered*.

---

## The flaw that survived, stated honestly

The chosen design puts the dead-man's switch **inside my own lab** (Grafana on `obs`). If the lab is
down, the alert can't fire — and that silence is invisible. The hosted heartbeat service (Attempt 2)
would have survived a lab outage; that's its genuine advantage. I traded it away for in-house
control, **with eyes open, and wrote the tradeoff down** rather than pretending the in-house version
was strictly better.

---

## Bonus gotchas this project surfaced (all generic, all real)

- **macOS `launchd` + Full Disk Access.** A `launchd` job that touches protected data needs FDA
  granted to the **exact** interpreter binary. A language-runtime upgrade **silently invalidates** the
  grant, and re-granting is **GUI-only** (you can't do it over SSH). Same trap twice: the fix looks
  done but delivers nothing.
- **A `LaunchAgent` only runs while a user is logged in.** If the monitor must survive logout, it has
  to be a `LaunchDaemon` (which needs admin). Otherwise a logout reads as "host down."
- **Some language runtimes don't use the OS trust store.** A from-python.org Python, for instance,
  ships its own CA bundle and won't see the system keychain — TLS calls fail with
  `CERTIFICATE_VERIFY_FAILED` until you install its certs, and **again after every upgrade.** Left
  uncaught, *every* alert would fail silently — the exact opposite of the goal.
- **Operational text is adversarial input to parsers.** An email *about* this monitoring job got
  ingested by the very calendar-parsing job it described, because an IP address (`10.0.0.30`) looked
  enough like a date to trip a lenient date gate. If your automation parses free text, make it
  categorically ignore messages about your own tooling.

---

## Takeaways

- **Match the tool to the job class.** Machine-level batch → textfile collector. Service-level →
  Pushgateway. "Did it run at all?" → a heartbeat/dead-man's-switch monitor.
- **A heartbeat turns silence into signal.** It's the cheapest, highest-value part.
- **Don't host the outermost health check inside the thing it's checking** — or if you do, know and
  document that you did.
- **Verify delivery on the real device**, not with a synthetic status code.
