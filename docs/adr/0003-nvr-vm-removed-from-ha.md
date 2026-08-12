# ADR-0003: NVR VM removed from HA management after a cross-node reset fault

## Status

Accepted — 2026-07-20

## Context

The cluster had a recurring and expensive fault: **taking one node down for maintenance would
reset the other one.** Planned work on one host produced an unplanned outage on the survivor —
including, because the firewall runs as a guest on this cluster, the network itself.

The obvious suspect was quorum. With two votes and no witness, losing one node left the survivor
inquorate, and that was genuinely a problem — it is fixed separately in
[ADR-0001](0001-external-quorum-witness.md).

But quorum loss alone does not reset a host. It stops HA resources from starting. Something was
turning "I have lost quorum" into "I must reboot now."

That something was the **HA watchdog**. Proxmox HA arms a watchdog timer on any node running
HA-managed guests. If the node cannot confirm quorum before the timer expires, it **fences
itself** — a hard reset — to guarantee that a guest cannot be running in two places at once.
That behavior is correct and is the entire point of fencing.

The node in question hosted the NVR VM under HA management. So: node A goes down for
maintenance → node B loses quorum → node B's watchdog cannot confirm quorum → node B resets
itself. Working as designed, and destructive.

The detail that made this avoidable: **the NVR VM cannot migrate.** It uses a raw disk
passthrough to a large local array on that specific host. It is pinned there by physics. HA
could never have restarted it anywhere else.

So the node was fencing itself to protect a guarantee that was meaningless for the only guest
that triggered it.

## Decision

Remove the NVR VM from Proxmox HA management entirely. Leave it as a normal, unmanaged guest
pinned to its host.

## Consequences

- The self-fencing trigger is gone. Losing one node no longer resets the other — which,
  together with ADR-0001, closed the fault completely.
- The NVR VM will **not** restart automatically after a host failure. It must be started by
  hand. Accepted: it could never have restarted elsewhere anyway, so the practical loss is a
  manual start rather than an automatic one.
- HA management is now reserved for guests that can actually migrate. A guest pinned to local
  hardware gains nothing from HA and can cost the whole node.
- Camera recording has a gap during a host outage. Understood and accepted for this workload.

## Alternatives considered

**Fix quorum only, and leave the VM under HA.** This was the tempting answer, and it is
incomplete. A quorum witness prevents the *common* trigger, but the node would still fence
itself in any future quorum event, protecting a guest that cannot move. Both changes were made;
neither alone is the fix.

**Disable the watchdog.** Rejected. The watchdog is what prevents split-brain, and disabling
it to stop an unwanted reset removes a safety mechanism instead of removing the reason it fires.
Treating the symptom.

**Move the VM to shared storage so it can migrate.** Rejected — the raw passthrough to local
disk is the reason the VM is configured this way. Recording surveillance video across the
network to shared storage, to gain an HA capability for a workload that tolerates a gap, is
worse on every axis.

**Migrate the NVR to a container or a different host.** Deferred, not rejected. Reasonable in
principle, but a larger change than the fault required, and this fault needed to stop happening.

---

## Note on the diagnosis

The original hypothesis — "two-node cluster, no quorum device" — was correct about a real
deficiency and still would not have explained the resets on its own. Quorum loss stops
services; the watchdog reboots hosts. Fixing only the thing that is obviously wrong is how
you end up with a fault that comes back later wearing different clothes.
