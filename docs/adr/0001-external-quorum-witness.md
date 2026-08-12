# ADR-0001: External quorum witness instead of a third cluster node

## Status

Accepted — 2026-07-20

## Context

The Proxmox cluster has two physical members. A two-node cluster cannot form a majority: with
two votes and a threshold of two, losing either node drops the survivor to 1/2 and it becomes
inquorate. An inquorate node will not start HA resources and, if it is running HA-managed
guests, will fence itself.

Two votes is not a cluster. It is two machines that both stop when either one does.

The standard answer is a third voting member. The cheap version of that answer — widely
recommended, and the one originally planned here — is a Raspberry Pi running `corosync-qnetd`
purely to hold a vote.

A third host already existed on the network: the file and backup server, which is deliberately
not a cluster member (see [ADR-0002](0002-file-server-outside-cluster.md)) but is always on,
on the same switch, and already carries the backup datastore.

## Decision

Run `corosync-qnetd` on the existing file/backup server as an **external quorum device**, not
as a cluster node.

A QDevice votes but does not join the cluster: it runs no guests, holds no cluster
configuration, and cannot have workloads scheduled onto it. Cluster state is now expected votes
3, total 3, threshold 2, flags `Quorate Qdevice`.

## Consequences

- Losing either Proxmox node leaves 2/3 votes. The cluster stays quorate and the survivor keeps
  running guests. This is the failure mode the change existed to fix.
- Losing the file server also leaves 2/3 votes — quorate, but with **zero remaining fault
  tolerance** until it is repaired. That host is now load-bearing for three separate things
  (bulk storage, backup datastore, quorum), which is a concentration worth being explicit about.
- No new hardware, no new operating system to patch, no additional power draw. Cost: $0.
- Any host running a quorum witness must be genuinely always-on. A machine that is rebooted
  casually becomes a source of quorum events rather than a cure for them.
- ⚠ **Verified by configuration, not yet by a real node failure.** The state flags are correct
  and the arithmetic is sound, but until a node is actually pulled this is a tested config and
  an untested outage. Recorded here rather than quietly assumed.

## Alternatives considered

**A third Proxmox node.** Correct, and the textbook answer. Rejected on cost and on scope: a
third compute member means another server, another PVE install to patch, and more power for
capacity that is not needed — the constraint in this lab is fast storage, not cores.

**A Raspberry Pi as a dedicated QDevice.** The original plan, and a common recommendation.
Rejected once it was clear an always-on host with better uptime characteristics already existed.
The Pi would have added a purchase, an SD card as a new failure point, and another OS to
maintain, to do a job an existing machine could do for nothing.

**Two-node with `two_node: 1` / `wait_for_all: 0`.** Lets a lone survivor stay quorate by
lowering the bar. Rejected: it trades split-brain protection for availability, and split-brain
in a cluster with shared storage is how you corrupt data rather than merely lose it.

**Accepting the two-node behavior.** Rejected — this was the status quo, and it produced the
fault documented in [ADR-0003](0003-nvr-vm-removed-from-ha.md).
