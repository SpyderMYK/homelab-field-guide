# Case study: a host that answers ping but drops every TCP connection

A debugging walkthrough. The value here isn't the fix — it's the *order of elimination*, and
how a wrong-but-plausible theory kept getting refuted by the next measurement until only the
truth was left.

All addresses/hostnames are examples. Cast:

- `obs` — an always-on host (`10.0.0.20`).
- `edge` — the router/firewall and default gateway (`10.0.0.1`), running a **stateful** firewall.
- `target` — the misbehaving machine (`10.0.0.30`). It's also reachable via a mesh VPN.

---

## Symptom

From `obs`, `target` **pings fine** (sub-millisecond, no loss) but **every TCP port times out** —
SSH, everything. Not "connection refused" (fast RST) — a full *timeout*. Yet `target` was
reachable over the mesh VPN the whole time.

First instinct: "the firewall on `target` is blocking inbound." Reasonable. Wrong. Here's how each
theory died.

---

## Elimination

**Theory 1 — host firewall blocking inbound.**
Checked the host firewall on `target`: **disabled**. Checked its packet filter (`pf`): enabled, but
the ruleset had nothing that would block inbound TCP on the LAN interface.
*Refuted, but weakly — kept going.*

**Theory 2 — packet filter drops it anyway.**
Ran a controlled test: **disabled `pf` entirely** for a fixed window while hammering the port from
`obs`. Still dropped, on every probe inside the window. Turning the filter off changed nothing.
*Refuted.*

**Theory 3 — it's the mesh VPN / route hijacking.**
The mesh VPN can, if misconfigured, pull a subnet's traffic through the tunnel. Checked routing
tables on both ends and the VPN's advertised routes: **clean**. The LAN subnet was on the physical
interface with the correct next-hop on both hosts. No tunnel route for the local subnet.
*Refuted — and this is the point where I stopped theorizing and started capturing.*

---

## The measurement that cracked it

Rule: when theories keep dying, **stop guessing and watch the packets.**

Packet capture on `target`'s LAN interface while `obs` opened a connection:

```
obs.61989   > target.22 : Flags [S]      ← SYN arrives at target
target.22   > obs.61989 : Flags [S.]     ← target SENDS a SYN-ACK
obs.61989   > target.22 : Flags [S]      ← obs retransmits (fresh timestamp)
target.22   > obs.61989 : Flags [S.]     ← target re-answers... forever
```

So `target` is **healthy**: it receives the SYN and answers. The handshake fails because `obs`
**never sees the SYN-ACK** and keeps retransmitting. Confirmed by capturing on `obs` too — its
interface shows the outbound SYNs and **zero inbound SYN-ACKs**.

Crucial detail from the same capture: **ICMP and the mesh-VPN's UDP flowed both directions the
whole time.** Only the **TCP SYN-ACK, `target → obs`,** vanished. A dumb switch can't be that
selective — only something L4-aware can. So the drop was *directional* and *L4-specific*.

## The detail that named the culprit

Re-ran the capture with **link-layer (MAC) addresses shown**. `target` was sending both its ICMP
echo-reply **and** its TCP SYN-ACK to the **gateway's MAC** (`edge`), not to `obs`'s MAC — even
though `obs` is on the *same subnet*.

That only happens if `target` has **no connected route for its own LAN subnet**. Its routing table
had host-scope `/32` entries for the gateway and itself, but the `10.0.0.0/24` connected route was
**missing**. So for any *other* LAN host, `target` fell through to the default route and shipped
replies to `edge`.

## Root cause

Asymmetric routing:

1. `obs → target` SYN goes **directly** over the LAN (obs has the connected route). `target`
   receives it.
2. `target → obs` reply is sent to the **gateway** (`target` lacks the connected route) →
   `target → edge → obs`.
3. `edge` is a **stateful firewall**. It never saw the forward SYN (that went direct), so the
   returning SYN-ACK is an **out-of-state TCP packet → dropped**.
4. **ICMP is effectively stateless** in that path, so ping sailed through — which is exactly why
   the symptom was "pings but no TCP."

Everything I'd suspected — host firewall, `pf`, the VPN — was innocent. The bug was a **missing
route**, and the firewall drop was a *downstream consequence* of the asymmetry, not the cause.

## Why the route vanished (the actual origin)

The mesh VPN client on `target` was running with **"accept advertised routes" enabled**, and at
some earlier point another node advertised the LAN subnet over the VPN. When the client installed
(and later withdrew) that route, it **clobbered the interface's connected route** for the subnet and
never restored it. The machine kept its IP; it just lost the local route.

## Fix

Two parts — restore now, prevent recurrence:

```sh
# 1. Restore the connected route
route add -net 10.0.0.0/24 -interface <lan-if>

# 2. Stop the VPN client from managing/clobbering LAN routes on a host that
#    is already physically on that LAN and doesn't need any accepted routes
<vpn> set --accept-routes=false
```

Verified the fix survives a VPN client restart (the event that used to break it) and, separately,
that a full reboot re-creates the connected route from DHCP and nothing removes it.

---

## Takeaways

- **"Pings but no TCP" is *most commonly* a host firewall silently dropping the SYN while allowing
  ICMP — rule that out first, as this story did.** Asymmetric routing is the sneaky runner-up when
  the firewall turns out innocent. The distinguishing tell isn't timeout-vs-refuse (both can time
  out — a firewall set to *drop* rather than *reject* also gives a timeout); it's the capture: a
  firewall drop means the SYN is never answered, whereas asymmetric routing means the host *does*
  emit a SYN-ACK that never reaches the client, with ICMP/UDP flowing fine and only return TCP dying.
- **When theories keep dying, capture packets.** The SYN-ACK-egresses-but-never-arrives observation
  ended three days of plausible guesses in one command. The MAC-level capture named the culprit.
- **A VPN client with "accept routes" on a host that's already on the advertised LAN is a foot-gun.**
  Don't accept routes for networks the host can already reach directly.
- **Verify the fix against the exact event that caused the break** (here: a VPN restart), not just
  "it works now."
