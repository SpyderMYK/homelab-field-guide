# Case study: auditing homelab storage against a real security standard

I ran my storage against **NIST SP 800-209** (*Security Guidelines for Storage Infrastructure*) —
not because a homelab needs to be compliant with anything, but because a good checklist finds the
things you've stopped seeing. The interesting part wasn't the standard; it was that **three of my
first "findings" were wrong once I actually checked**, and the two real problems were ones I'd have
sworn were fine.

Generic throughout. Cast: `nas` — a Linux box running a ZFS mirror plus the observability stack,
reachable on the LAN and over a mesh VPN.

---

## Scope it down first

Most of an enterprise storage standard doesn't apply to a homelab — SAN/Fibre-Channel zoning,
multi-tenant arrays, cloud object storage. Ignore those. What *does* map onto a small lab:

- **Access control** — who can reach the storage, and with what rights.
- **Data resilience** — surviving loss, not just disk failure (the revision that prompted this added
  a whole family here — ransomware/integrity is now first-class).
- **Audit / alerting** — do you actually find out when something breaks?
- **Media protection** — encryption at rest, safe disposal.

Four questions, not four hundred. That's the useful reduction.

---

## Three "findings" that were wrong (verify before you flag)

Recording these because they're the whole lesson: **an audit finding is a hypothesis until you've
checked it on the live system.**

1. **"SMB is wide open to guests."** The global config *looked* alarming (`map to guest = Bad User`,
   `usershare allow guests = yes`). But the actual share required a named user (`valid users = …`),
   confirmed with `testparm -s`, and there was exactly one Samba account. Not exposed.
2. **"Scrubs aren't automated."** The systemd scrub timers were disabled — but the distro ships a
   `cron.d` entry that runs a scrub on the second Sunday monthly. The last scrub I'd "caught" as
   suspicious was just that scheduled run.
3. **"The notification config is still the placeholder."** A quick grep matched the `CHANGE_ME`
   example file; the *live* config had a properly randomized value. Working fine.

Each took two minutes to check and would have been an embarrassing thing to "fix."

---

## Real problem #1 — an NFS export open to everything, used by nothing

```
/tank  <LAN-subnet>(rw,sync)  100.64.0.0/10(rw,sync)
```

`100.64.0.0/10` is the **entire mesh-VPN address range** — so *every* device on the VPN (phones,
tablets, a relative's laptop) could mount the whole pool **read-write**, on nothing more than an
IP match. Plus the entire LAN.

Then the part that made the fix trivial: **nothing was using it.** Verified via the authoritative
NFSv4 client list (`/proc/fs/nfsd/clients` — empty), `showmount -a` (empty), zero established
connections on `:2049`, and no mounts in two weeks of the server's journal. So it was pure attack
surface with **zero benefit**, and removing it was **zero-risk**.

**Fix:** unexport, disable the NFS server, and confirm from *another* host that `:2049` now refuses.
(SMB, which was actually in use and properly authenticated, stayed.) Also disabled the now-orphaned
`rpcbind` — no NFS, no reason for it to listen.

**Lesson:** audit not just *what's configured* but *what's actually used*. The safest thing to
remove is a service with a wide blast radius and no clients.

---

## Real problem #2 — alerting that had never delivered anything

The ZFS event daemon (**ZED**) was running and enabled — looked healthy. It was configured to email
`root`. But **no mail transport was installed at all** (no `sendmail`, no `postfix`, nothing). So
every ZFS event — including "a mirror disk is failing" — was being written to a mail spool that went
nowhere, silently. ZED doesn't log or fail when its mail program can't run; the alerts just evaporate.

**This is the single most dangerous kind of bug in a homelab: monitoring that looks configured and
delivers nothing.** A degraded mirror could have sat unnoticed until the *second* disk died.

**Fix:** modern OpenZFS has native push-notification support (`ZED_NTFY_*` in `zed.rc`) — point it at
the same push service the rest of the lab already uses, no mail transport required. Then:

- Turn on **verbose** notifications so you also get a message on *successful* scrub completion — a
  **heartbeat**, so that silence becomes meaningful rather than ambiguous.
- **Prove it end-to-end with a real event**, not a synthetic test: kick an actual `zpool scrub`, let
  ZED fire on completion, and confirm the notification lands on your phone. An HTTP 200 from the push
  service only proves it was *accepted*.

---

## Real problem #3 — a mirror is not a backup (the 3-2-1 gap)

The pool had a **mirror** (survives a disk) and **automatic snapshots** (survive deletion and
file-level ransomware). It did **not** have a copy anywhere else. A mirror + snapshots on one box
does not survive theft, fire, a failed PSU, or a root compromise that runs `zfs destroy` on the
snapshots too. This is exactly the "data resilience" gap the standard now treats as first-class.

The fix was mostly **triage**, and triage held a surprise:

- What looked like ~130 GB of stale "migration leftovers" was actually **irreplaceable single-copy
  personal data** — an old home directory, a phone photo dump, camcorder/360 footage. The source
  devices were gone. *Assume nothing about a directory named `restore/` until you look inside.*
- Conversely, a big chunk of the pool was **trivially replaceable** — tens of GB of game-capture
  video. Pruning that first **halved the backup set** before designing anything. Back up what's
  irreplaceable, not what's merely large.

**The off-box copy:** a dedicated external disk, written with `rsync`, then **verified by checksum**
(`rsync -n --checksum` re-reads both sides and reports any mismatch — matching file counts and byte
totals only prove nothing is *missing*, not that the bytes are *correct*). Kept **normally
unplugged** — for cold, static archives that's a free air gap and the best defense against the
compromise scenario that makes the data vulnerable in the first place.

---

## The ZFS gotcha that trips everyone

After pruning ~60 GB, `du` dropped but the **pool showed zero space reclaimed**. Snapshots pin
deleted blocks:

```
USED 128G   REFER 70G   USEDBYSNAPSHOTS 58G   ← the "deleted" data is held by snapshots
```

That's a *feature* — the delete is reversible until the snapshots expire — but it means "I deleted
files, why is the pool still full?" has a specific answer. The space returns when the snapshots
holding those blocks age out (or you destroy them deliberately). Here it didn't matter (the pool was
14% full); the point of the prune was a smaller *backup set*, which it achieved regardless.

---

## Takeaways

- **A published standard makes a great checklist — after you scope it to your reality.** Four
  questions (access, resilience, audit, media) covered everything that mattered.
- **An audit finding is a hypothesis. Check it live.** Three of mine were wrong; the real problems
  were elsewhere.
- **Remove wide-open services that nothing uses** — biggest risk reduction for the least effort.
- **The most dangerous monitoring is the kind that silently delivers nothing.** Add a heartbeat and
  test the full chain with a real event on your real device.
- **A mirror and snapshots are not a backup.** Get one copy off the box, prefer it offline, and
  **verify it by reading it back**.
- **Bind storage/monitoring services to the interface that needs them** — a service on `0.0.0.0`
  with no auth (NFS, `node_exporter`) is reachable by every device on that network.
