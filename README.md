# A Field Guide to a Small, Serious Homelab

Notes from running a small home lab as if it were production — enough to actually learn
the patterns that matter (monitoring, backups, network segmentation, incident response),
without pretending to be a datacenter.

This is opinionated and lesson-driven. Every "rule" below is here because ignoring it
cost me something. All addresses, hostnames, and topology are **generic examples** — swap
in your own.

> Scope: a handful of always-on machines (mini PCs, a NAS box, some SBCs), one edge
> firewall/router, and a mesh VPN for remote access. If you run a rack, you already know
> most of this.

---

## Ten principles that held up

**1. Verify against the live system; don't trust the doc (including your own).**
State drifts. A wiki page that said "this is configured" was wrong often enough that I now
treat every doc as a *claim* to re-check, not a fact. Where a note says "verified," it means
someone looked at the running system on a specific date — and even that decays.

**2. Silence must be meaningful.** An alerting path you never exercise is indistinguishable
from a broken one. I've shipped two "working" alert setups that delivered *nothing* — one
emailed a local user with no mail transport installed, another posted to a notification topic
nobody was subscribed to. Both looked configured. Fix: send a **daily heartbeat** on the same
path as your alerts, so that *absence* is itself a signal, and test the full chain with a real
event, not a synthetic "HTTP 200."

**3. Don't host your monitor inside the thing it monitors.** A dead-man's switch that runs on
the same box (or same lab) it's watching has a bootstrap flaw: when the thing dies, so does the
detector, and the silence is invisible. Put the outermost health check somewhere independent.

**4. A mirror is not a backup. A snapshot is not a backup.** RAID/ZFS mirrors survive a *disk*
failure. Snapshots survive *deletion*. Neither survives the box being stolen, catching fire, or
a root compromise that runs `zfs destroy`. Aim for **3-2-1**: three copies, two media, one
off-site/offline. An occasionally-attached external disk that's normally unplugged is a
legitimate — and ransomware-resistant — third copy.

**5. Verify backups by reading them back.** "The copy finished" is not "the copy is correct."
Compare checksums end-to-end (e.g. `rsync -n --checksum`), which re-reads both sides. Count
files and bytes. A backup you've never restored (or at least checksummed) is a hope, not a
backup.

**6. Trim attack surface you're not using.** The worst exposure I found in my own lab was an
NFS export shared read-write to an entire subnet *and* my whole mesh VPN range — used by exactly
zero clients. Unused services, wide-open exports, and services bound to `0.0.0.0` that only need
to be local are free risk. Audit what's *listening* and who can reach it.

**7. Segment, and keep the boundary honest.** Put untrusted/IoT devices on their own segment.
Keep your lab separate from the household network. Then periodically confirm the segmentation
still holds — it's easy to accidentally bridge two segments (e.g. a dual-homed host with a leg
on each), which quietly defeats the firewall between them.

**8. Treat inventory as a source of truth, and generate from it.** A single place that records
"what runs where, on what port, reachable how" pays for itself the first time you debug at 1 a.m.
Better still, *generate* downstream config (dashboards, DNS, monitoring targets) from that source
so it can't drift.

**9. Prefer the boring, upstream-recommended tool.** When a design fights the documentation,
the docs usually win. Reaching for a fancier component than the job needs (see the batch-metrics
example below) adds failure modes for no benefit.

**10. Write it down as you go.** Session notes — what you changed, why, what you *rejected* and
why, and the exact revert steps — are worth more than polished docs written later. Include the
dead ends; "we tried X, it failed because Y" saves the next person (often future-you) the detour.

---

## A generic reference architecture

Roles, not products. One machine can wear several hats when you're small.

```
                        Internet
                           │
                    ┌──────┴───────┐
                    │  edge router │  firewall · DHCP · DNS · NAT
                    │  / firewall  │  (e.g. an OPNsense/pfSense/OpenWrt box)
                    └──────┬───────┘
              LAN 10.0.0.0/24  │   (segment your IoT/guest onto separate VLANs)
        ┌──────────────┬───────┴────────┬───────────────┐
        │              │                │               │
   ┌────┴────┐   ┌─────┴─────┐    ┌─────┴─────┐   ┌──────┴──────┐
   │ obs-host│   │  nas-host │    │  app-node │   │  sbc-fleet  │
   │ metrics │   │ ZFS mirror│    │ services  │   │ (Pi-class)  │
   │ logs    │   │ + backups │    │ / compute │   │ sensors etc │
   └─────────┘   └───────────┘    └───────────┘   └─────────────┘

   Remote access: a mesh VPN (e.g. Tailscale/WireGuard) — NOT port-forwarding.
```

- **Edge**: one device owns routing, DHCP, DNS, and NAT. Enable NAT in exactly one place
  (double-NAT causes subtle breakage).
- **Observability host**: Prometheus (metrics) + Grafana (dashboards/alerts) + a log store
  (Loki). Scrapes everything else.
- **Storage host**: ZFS mirror, scheduled scrubs, snapshots, and an off-box copy.
- **App/compute node**: whatever you're actually running (containers, a local LLM, game servers).
- **Remote access**: a mesh VPN gives you encrypted, no-inbound-ports reachability to any node
  from anywhere. Vastly safer than exposing services to the internet.

---

## Observability starter kit

The minimum that makes a lab debuggable:

1. **`node_exporter` on every host**, scraped by Prometheus. Host CPU/mem/disk/net + the
   invaluable `up{}` (is this host even reachable?).
2. **Grafana** for dashboards and **unified alerting** (you don't need a separate Alertmanager
   to start). Provision dashboards/alerts from files checked into git.
3. **Loki + a log shipper** (Alloy/Promtail/Vector) so logs are searchable in one place. Point
   your firewall's syslog here too.

**Batch/cron jobs are the exception.** They have no scrape endpoint — they just run (or don't).
Two right answers:

- **Machine-level job** (bound to one host): write a metric to a `.prom` file and let
  `node_exporter`'s **textfile collector** pick it up. Write it **atomically** (temp file in the
  same dir, then `os.replace`) or the exporter will read a half-written file. This is the
  Prometheus-recommended pattern — *not* Pushgateway, whose own docs say it's only for
  service-level jobs "not tied to a specific machine," and which keeps stale metrics forever (no
  TTL).
- **"Did this run at all?" (dead-man's switch)**: a heartbeat/ping monitor
  (`healthchecks.io`-style, self-hostable) that alerts on *absence*. Keep it independent of the
  lab (principle #3).

For alert delivery, a push-notification service (e.g. `ntfy`) is a low-friction target that
reaches your phone. **Treat any unauthenticated topic/URL as a secret** — on a public instance
the random name *is* the access control.

---

## Storage that you can actually trust

- **ZFS mirror** for disk-failure survival and end-to-end checksums (it catches silent bit rot;
  most filesystems don't).
- **Scheduled scrubs** (monthly is fine for a small pool) — and make sure the scrub *result* is
  alerted. ZFS's event daemon (**ZED**) can notify on pool degradation; confirm it actually
  delivers (it often defaults to mailing `root`, which goes nowhere without a mail transport).
- **Automatic snapshots** (frequent/hourly/daily/weekly) for oops- and ransomware-recovery at the
  file level.
- **An off-box copy** — the part everyone skips. Cold, static archives (photos, old projects) are
  perfect for a periodically-attached external disk. Keep it normally unplugged: that's a free
  air gap.
- **Reclaiming space after a delete?** Remember snapshots pin deleted blocks — you won't get the
  space back until the snapshots holding them expire. That's a feature (the delete is reversible),
  not a bug.

---

## Networking gotchas that cost me hours

- **A host that answers ping but drops all TCP** is *most often* a host firewall silently dropping
  the SYN while letting ICMP through — check that first. But it can also be **asymmetric routing**,
  which is subtler: if a machine loses the connected route for its own subnet it sends replies via
  the gateway, and a *stateful* firewall then drops the "out of state" return packets while
  stateless ICMP sails through. Tell them apart with a packet capture on both ends — a firewall
  drop means the SYN is never answered; asymmetric routing means the host *does* emit a SYN-ACK
  that never reaches the client (see the [case study](case-studies/host-pings-but-drops-all-tcp.md)).
- **Don't advertise, over your mesh VPN, a subnet that your VPN peers physically live on.** It
  creates exactly the asymmetry above and breaks LAN connectivity between those peers. Advertise
  routes only for networks your remote peers *can't* otherwise reach.
- **Give services stable names, not memorized IPs.** Per-host DNS overrides on your resolver (or
  the mesh VPN's MagicDNS) make everything legible and survive re-addressing.
- **macOS as a lab server has sharp edges.** `launchd` jobs that need Full Disk Access must have
  it granted to the *exact* binary — a runtime upgrade silently invalidates the grant, and
  re-granting is GUI-only. A `LaunchAgent` only runs while a user is logged in; use a
  `LaunchDaemon` if it must survive logout. And a headless Mac has its own quirks around sleep,
  screen sharing, and boot-on-power.

---

## Small operational habits that punch above their weight

- **Back up before you edit; note the backup path in the change.** `cp file file.bak-$(date +%s)`
  costs nothing and turns "oops" into "revert."
- **Diff before you restore.** Confirm the only delta is the one you intend.
- **Verify from the outside.** After closing a port, check it's refused *from another host*, not
  just "the service stopped."
- **Beware autocorrect in the loop.** Pasting commands/configs through a mail client or chat can
  silently turn `--flag` into an em-dash or capitalize `curl`. Move exact text via the clipboard
  and re-check it.
- **Keep secrets out of git from commit #1.** A `.gitignore` that excludes config backups, keys,
  `.env`, and session data belongs in the *first* commit — history is forever, and a secret
  committed once is committed always.

---

## Deep-dive case studies

Longer walkthroughs of two problems from the notes above — the full reasoning, including the
wrong turns:

- **[A host that answers ping but drops every TCP connection](case-studies/host-pings-but-drops-all-tcp.md)**
  — a debugging story in order-of-elimination form. Three plausible theories die one by one until a
  packet capture reveals asymmetric routing, and a MAC-level capture names the cause.
- **[A dead-man's switch for a remote scheduled job](case-studies/dead-mans-switch-for-a-remote-cron-job.md)**
  — a design journey through three approaches (Pushgateway → hosted heartbeat → node_exporter
  textfile collector), why the first two were rejected, and the logout-tolerant alert query that
  makes it robust.
- **[Auditing homelab storage against a real security standard](case-studies/auditing-homelab-storage-against-nist.md)**
  — using NIST SP 800-209 as a checklist: three "findings" that were wrong once verified, an NFS
  export open to everything and used by nothing, alerting that had never delivered, and the 3-2-1
  gap behind a ZFS mirror.

---

## Tooling I reach for (all free / open-source)

| Need | Tool(s) |
|---|---|
| Edge firewall/router | OPNsense / pfSense / OpenWrt |
| Metrics | Prometheus + node_exporter |
| Dashboards + alerts | Grafana |
| Logs | Loki + Alloy (or Promtail/Vector) |
| Inventory / source-of-truth | NetBox |
| Remote access | Tailscale / plain WireGuard |
| Storage | ZFS (mirror + snapshots + scrub + ZED) |
| Push notifications | ntfy |
| Cron/heartbeat monitoring | healthchecks.io (hosted or self-hosted) |

---

## License & contributing

Copyright © 2026 SpyderMYK. Licensed under **[CC BY 4.0](LICENSE)** (`SPDX-License-Identifier:
CC-BY-4.0`) — use it, adapt it, even commercially; just give attribution:

> "A Field Guide to a Small, Serious Homelab" by SpyderMYK (https://github.com/SpyderMYK), CC BY 4.0.

Corrections and additional war stories are welcome via issues or pull requests.

---

*Written from real experience running a small homelab; all specifics genericized. If a section
saved you a debugging session, that's the whole point.*
