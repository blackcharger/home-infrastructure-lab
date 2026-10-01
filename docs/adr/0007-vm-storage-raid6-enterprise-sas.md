# ADR-0007: Replace consumer SATA VM storage with RAID6 on used enterprise SAS SSDs

## Status

Accepted — 2026-09-30

## Context

Every VM on the main compute node lived on one consumer QLC SATA SSD behind the node's hardware
RAID controller. The documentation said the node had **two 1 TB SSDs in RAID1**. That was never
true. When the controller was finally queried during an incident, it showed **two separate
single-drive RAID0 volumes**: one holding the hypervisor OS, the other holding every VM. There
was no redundancy anywhere, and the documentation had been asserting there was for months.

The VM drive then failed in a pattern that got steadily worse:

1. **Link drops under sustained load.** Three times, the controller lost the drive mid-way
   through the nightly backup's full-disk read and took the volume offline. Every VM on the node
   froze. Recovery each time meant importing the drive's foreign configuration and letting the
   controller flush its pinned write cache back to disk. No data was lost, because the cache
   was never discarded.
2. **Unrecoverable medium errors.** By the third drop the drive also had sectors it could not
   read at all. That meant actual flash failure, not just a flaky link.

Consumer SATA SSDs dropping their link behind a server RAID controller under sustained I/O is a
known failure mode. Backups were disabled to stop provoking it, which left the node with no
current VM backups at all.

## Decision

Move all VM storage to a new **RAID6 array of eight used enterprise SAS SSDs**: 400 GB,
12 Gb/s SAS, write-intensive, with power-loss protection. The array uses the controller's
battery-backed write-back cache. Migrate VMs with as little downtime as possible, keep the old
drive untouched until the new array has run cleanly for a burn-in period, then retire it under
warranty.

- **SAS, not SATA.** SAS drives are what this controller and backplane are built for. Enterprise
  SAS drives also have power-loss protection, which consumer drives don't.
- **RAID6, not RAID10.** The usual advice is RAID10 for VM storage. That advice comes from
  spinning disks, where RAID6's write penalty of six I/Os per small random write is crippling.
  It doesn't hold here. These SSDs each sustain tens of thousands of random IOPS. The write-back
  cache absorbs the parity work. A 400 GB SSD rebuilds in minutes, not hours. RAID6 gives
  **50% more usable space (≈2.2 TiB vs ≈1.5 TiB)** and survives **any** two drive failures,
  where RAID10 survives only some combinations. That matters because of the next point.
- **Used drives, inspected before use.** All eight came from one source server: ~6.2 years
  powered on, 16–18% of rated endurance used, zero grown defects, zero uncorrected errors, same
  firmware. Identical age means **correlated failure risk**, and RAID6's tolerance for any two
  failures is the direct answer to it. The drives came with a 90-day return window, so the
  pre-use inspection had teeth.
- **Migrate first, salvage second.** Nine VMs moved with the hypervisor's disk-mirror job, six of
  them while running, with the old copies kept. Two could not: the mirror job aborts on the first
  unreadable block. Those were stopped and copied with `ddrescue`, which recovers every readable
  block and zero-fills the rest. Each unreadable region (128 KiB and 256 KiB) was then traced
  from controller LBA → thin-pool data block → guest filesystem block → file. That turned "some
  data is lost" into a named list of affected files. One hit was free space plus two unused
  kernel-header files. The other hit one database file, which can be restored from backup.

## Consequences

- **≈2.2 TiB of redundant VM storage** replaces a single unprotected drive. Any two of the
  eight drives can fail without data loss.
- Backups were re-enabled once the VMs were off the failing drive. The first run after
  migration is a full read, because every disk changed.
- **The hypervisor OS still sits on a single consumer SSD**, the same model as the one that
  failed. It's now the only non-redundant storage on the node, and it's the next thing to fix.
- **Monitoring gap found.** The out-of-band controller's health rollup reported the failing drive
  as *healthy* while it had six media errors on record. Redfish health only flags drives that are
  dead or predicting failure. A RAID array of same-age drives needs alerting on rising error
  counts read directly from the RAID controller, before the first drive dies.
- The failed drive stays installed, with every old VM copy on it, until the new array has proven
  itself. Then it's wiped and returned under warranty: well inside both its three-year term and
  its written-bytes limit.

## Alternatives considered

**Two enterprise SATA SSDs in RAID1.** The first plan. Simple, but it keeps the SATA path that had
just failed and survives only one drive failure.

**RAID10 across the same eight drives.** Faster small writes and simpler rebuilds. Rejected: on
SSDs behind write-back cache the speed difference is invisible at this workload, and it gives
up a third of the capacity and the any-two-failures guarantee.

**Fewer, larger used SSDs (4 × 1.6 TB) from an auction.** Best cost per terabyte on paper, but the
price climbed past budget, and an auction gives no delivery date while the VM drive is failing.

**Four larger write-intensive SSDs (4 × 800 GB).** Same usable capacity as RAID10 on eight smaller
drives, at nearly double the cost per terabyte, for endurance this workload will never use.

**10K RPM SAS hard drives.** Cheap and plentiful. Rejected: roughly 200 random IOPS per drive is a
large regression for VM storage, and the planned Kubernetes control plane needs low fsync latency.

---

## The generalization

The most damaging fact here was not the failing drive. It was **"RAID1" written in the
documentation for a configuration that never existed.** Every decision built on that line
(acceptable risk, backup scheduling, what a drive failure would cost) inherited a redundancy that
wasn't there. The fix for the drive was eight new SSDs. The fix for the documentation is a habit:
**claims about the storage layer get verified against the controller, not copied from notes.**
