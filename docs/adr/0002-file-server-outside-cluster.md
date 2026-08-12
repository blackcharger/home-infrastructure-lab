# ADR-0002: File server deliberately excluded from the cluster

## Status

Accepted

## Context

The lab runs three enterprise servers. Two are Proxmox VE cluster members providing compute.
The third — the largest machine by storage, with a 26-bay chassis, a hardware RAID controller
with battery-backed write cache, and by far the most RAM-per-dollar — is **not** a cluster
member. It runs plain Ubuntu.

To anyone reading the hardware list, this looks like an oversight. The biggest box is sitting
outside the cluster. That is the entire reason this record exists.

The pull toward adding it is real: more cores in the pool, more RAM, and a third native vote
that would remove the need for [ADR-0001](0001-external-quorum-witness.md) entirely.

## Decision

Keep the file/backup server **outside the Proxmox cluster**, running Ubuntu, serving three
roles: bulk storage, backup datastore host, and external quorum witness.

## Consequences

- **The backup target is not a member of the thing it backs up.** A cluster-wide problem —
  a bad upgrade, a corosync misconfiguration, a fencing storm — cannot take the backup datastore
  with it. This is the single strongest argument for the arrangement, and it is worth more than
  the cores it costs.
- Its storage array is presented to the cluster over the network rather than as cluster-managed
  storage. Simpler to reason about, and the file server can be rebooted without the cluster
  treating it as a storage failure.
- It can be maintained on its own schedule. It is not subject to cluster-wide upgrade ordering
  or corosync version compatibility.
- **Cost, stated honestly:** its CPU and RAM are unavailable to the compute pool. On paper that
  is the most capable host in the lab sitting outside it. Accepted deliberately — the binding
  constraint here is fast storage, not compute, and the cluster is not core-starved.
- It becomes a concentration point: storage, backups, and quorum all depend on one machine. See
  ADR-0001's consequences. This is the main risk the decision creates.

## Alternatives considered

**Add it as a third Proxmox node.** Would supply a native third vote and add its resources to
the pool. Rejected: it puts the backup datastore inside the failure domain it is meant to
survive, and it converts an independently maintainable storage host into a machine bound to the
cluster's upgrade and quorum lifecycle. The vote problem is solvable for $0 without that
coupling — which is what ADR-0001 does.

**Run it as a node but exclude it from HA and scheduling.** Achieves the vote and keeps
workloads off it. Rejected as the worst of both: still coupled to cluster upgrades and corosync,
still inside the blast radius, without the resource benefit that motivated joining.

**Move bulk storage into the cluster and repurpose the chassis.** Rejected — the RAID controller
with battery-backed write cache and 26 populated bays is the reason this host exists. Breaking
that up to gain compute the lab does not need is a bad trade.
