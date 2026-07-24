# Case study: rolling out fleet-wide observability — and what the rollout revealed

This started as a boring **accuracy pass** — reconcile the monitoring docs with reality — and grew,
the way these things do, into "wait, half the fleet isn't actually being scraped." The lessons
worth keeping are less about the tools (Prometheus + Grafana + Loki + node_exporter, all standard)
and more about the **deployment mechanics**, the **gotchas that eat an afternoon**, and the fact
that *doing the rollout surfaced real gaps I'd have never gone looking for*.

Generic throughout. Cast: `obs` (the box running the observability stack in containers, and a ZFS
pool), `ops` (the box that runs the backup/sync cron jobs), and a mixed fleet — Linux SBCs, a couple
of headless Macs, and the firewall.

---

## Get an exporter on every host — the unglamorous part that matters

Coverage is binary per host: either you can see it or you can't. The audit found the target list had
**stale labels** and **missing hosts** — two machines simply weren't being scraped, so they were
invisible. Closing that is 80% of the value.

Deploying `node_exporter` across a heterogeneous fleet means three different patterns:

- **Linux** — the distro package + a systemd unit (`enabled` + `active`). Boring, correct, done.
- **The firewall** — a first-party node_exporter plugin, enabled with the collectors you want.
- **macOS** — this one hid a real bug. One Mac had node_exporter running as a **user LaunchAgent**.
  It worked... until you realized the box has no auto-login, so a **reboot would silently kill it**
  and never bring it back. Another Mac had it installed but not running at all (that's why its target
  was down). The fix on both: run it as a **root `LaunchDaemon`**, which starts at boot without a
  user session.

**Lesson:** for any always-on service on a headless machine, "it's running now" is not "it survives a
reboot." On macOS specifically, **LaunchAgent = only while logged in; LaunchDaemon = at boot**. Verify
the survive-reboot property explicitly (`netstat -an | grep <port>` after an actual reboot), don't
assume it.

---

## The gotcha that ate an hour: single-file Docker bind mounts pin the inode

The stack's configs (`prometheus.yml`, the Loki config, the log-shipper config) were bind-mounted as
**single files** into their containers. I edited one with `sed -i`, sent the container a reload
signal, watched it report success... and the change didn't take.

Cause: `sed -i` (and most editors) don't edit in place — they **write a new file and rename it over
the old one**, which gives it a **new inode**. A single-file bind mount is bound to the *original
inode*; the container keeps reading the old file. Worse, the reload signal **succeeds** — it re-reads
the stale content and looks healthy.

**Fix:** after editing a single-file bind mount, `docker restart <container>` (or mount the *directory*
instead of the file, so a rename inside it is visible). This one is subtle precisely because the
reload *appears* to work.

---

## Config-as-code: one source of truth, everything generated

The change that made the whole thing maintainable: a **single inventory** (what exists, where it runs,
on what port, reachable how) as the source of truth, and *generate* the downstream config from it:

- **Prometheus scrape targets** — from the inventory, not hand-edited.
- **DNS** — every host and service gets a name from the same data.
- **A dashboard/homepage** — tiles generated from the inventory on every sync.

Onboarding a host or service becomes one action: **add it to the inventory, run the sync.** Targets,
DNS, and dashboard all follow. Nothing drifts because nothing is maintained in two places.

A nice refinement: make **service DNS follow the service, not the host**. `grafana.<lan>` should point
at *whichever* host currently runs Grafana. Then moving a service to a different box is a one-field
change in the inventory — the DNS record and every dashboard link update automatically, and no
bookmark breaks. (This also lets you retire per-host DNS hacks in favor of one DNS generator that owns
its records and refuses to touch hand-made ones.)

---

## A reality about labels: renames split your history

When you rename a host, Prometheus/Loki **historical series keep the old label**; only new data gets
the new one. That's not a bug to fix — it's how time-series labels work — but it means:

- Fix the label at the *source* (the exporter/scrape config and the log shipper), then
- Expect dashboards to show a seam at the rename, and write queries that tolerate both labels if you
  care about spanning it.

Hunt down *every* place the old name is baked in — scrape config, log-shipper config, any
generated-config script that hardcodes a hostname — or the old label quietly comes back on the next run.

---

## What the rollout surfaced (the real payoff)

Instrumenting everything made two problems visible that I wasn't looking for:

**A backup single-point-of-failure.** With everything now on one dashboard, it was obvious that *all*
backups landed on **one box's internal disk** and the NAS pool was receiving **nothing** — so that one
box was a single point of failure for all backup history. Fix: a nightly push of the backup set to a
dataset on the NAS, plus snapshots there. Ordering matters: local backup jobs → push to NAS → snapshot.

**Don't reinvent what the platform already does.** I wrote a tidy snapshot-rotation script for that new
NAS dataset... then discovered the box was *already* running an auto-snapshot service covering the whole
pool. I **deleted my script** and let the existing one own retention. A homelab accretes redundant
automation; a rollout is a good time to find and remove it.

Two supporting habits fell out of this:

- **Verify freshly-written ZFS copies by listing (file count/size), not `du`** — `du` lags newly
  written data and will lie to you right after a push.
- **Add canaries for the things that fail silently.** A health check that asserts "the newest backup
  *on the NAS* is less than 36h old" turns a silent backup-pipeline death into an alert — the same
  "silence must be meaningful" principle, applied to the backup path itself.

---

## Security aside: a rollout is also a good time to find forgotten exposure

While mapping what was reachable, I found a **public tunnel** still serving a stale page from months
earlier — gated by a "passcode" that turned out to be **client-side base64** (i.e. decorative; anyone
could read or bypass it). Turned it off, confirmed it was unreachable from the internet, and left the
private/VPN-only services untouched.

Related pattern worth stealing: where I *did* want a one-click "publish/unpublish" control, I fronted it
with a **forced-command SSH key** — the key's `authorized_keys` entry is pinned to a single script that
accepts only `on|off|status`, with no shell, no pty, no forwarding. Even a fully compromised control UI
can only toggle that one thing. Least privilege for the automation, not just the humans.

---

## Takeaways

- **Coverage is the point.** An unscraped host is an invisible host; close the gaps first.
- **"Running now" ≠ "survives a reboot."** Especially on macOS (LaunchAgent vs LaunchDaemon) — verify it.
- **Single-file Docker bind mounts pin the inode** — edits via rename don't reach the container, and the
  reload *looks* successful. Restart, or mount the directory.
- **Generate config from one source of truth.** Add-to-inventory-and-sync beats editing five files.
- **A cleanup/rollout pass reveals real gaps** — a backup SPOF, redundant automation, forgotten public
  exposure. Go looking while you're already in there.
