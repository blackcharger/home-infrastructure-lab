# ADR-0008: Move the firewall off the cluster onto dedicated hardware

## Status

Accepted — 2026-10-03. Implemented 2026-11-06.

## Context

The firewall/gateway (pfSense, also the WireGuard endpoint and DHCP server) ran as a VM on one
of the two Proxmox cluster nodes. That was cheap and it worked, but it tied two unrelated
things together: **every change to the cluster was also a change to the network.**

- Rebooting the node that hosted the firewall took the internet down for the whole house.
- The quorum witness from [ADR-0001](0001-external-quorum-witness.md) had never been tested under
  a real node failure, because the only honest test is to pull a node, and pulling that node cut
  the network the test depended on.
- Planned storage work on that node, such as replacing its single consumer boot SSD, meant an
  outage for everyone, not just for the lab.
- In an incident, the tools for fixing the cluster (remote consoles, the out-of-band management
  VLAN, the VPN) were routed by a VM running on the cluster being fixed. The fault this caused is
  in [ADR-0003](0003-nvr-vm-removed-from-ha.md): one node's maintenance reset the other, and the
  network went with it.

The firewall also held the node's only SFP+ ports. That kept the node's own management bridge on
a 1 GbE port, where migrating its VMs elsewhere takes about an hour instead of minutes.

The WAN is a **2 Gbps** cable service. Any replacement needed 10 Gb-capable ports. A firewall
with 1 GbE ports would cut the service in half.

## Decision

Run the firewall on a **dedicated, used 1U server** (Dell R230) outside the cluster:

- **Intel X520 dual-port 10 Gb SFP+** for WAN and a VLAN-trunked LAN. FreeBSD's Intel `ix` driver
  is the well-trodden path, and the copper WAN handoff uses a rate-adapting 10GBASE-T module.
- **Two used enterprise SATA SSDs (with power-loss protection) in a ZFS mirror** for boot.
- **Out-of-band management** (iDRAC Enterprise) plus a local monitor and keyboard, so the box can
  be rescued with no working network.
- **Move the existing configuration, don't rebuild it.** Export the VM's config, rename the
  virtual interfaces to the physical ones, and **clone the VM's WAN MAC**, so the ISP sees the same
  device and the public IP doesn't change. Firewall rules, VLANs, DHCP reservations and VPN peers
  carry over unchanged.
- **Keep the old VM as a cold standby.** Stopped, with start-at-boot off, for at least a week
  after cutover. Rollback is moving two cables and starting a VM.

Total cost was under $250 in used parts.

## Consequences

- **Cluster work is no longer network work.** Either Proxmox node can be rebooted, drained or
  failure-tested without touching the internet, the VPN or the management VLAN. The quorum
  failure test, still outstanding, can now be run.
- **The firewall is now a single, separate failure domain.** It has one power supply and no
  standby that starts on its own. Losing it is an outage until the cold standby is started by
  hand. It was always a single point of failure. The difference is that it no longer fails *with*
  the cluster.
- **Throughput at the full 2 Gbps hasn't been load-tested yet.** The entry-level CPU was judged
  sufficient for routing at this speed. The socket takes a Xeon if IDS/IPS or a faster WAN ever
  needs the headroom.
- **Packages don't come with the config.** The restored configuration still listed the WireGuard
  and sudo packages, but the box booted while the WAN held a stale DHCP lease from its staging
  address. The automatic reinstall failed with no visible error. Both were installed by hand after
  the WAN came up, and the tunnel config was intact. Verify the installed packages after any
  restore. Don't assume they reinstalled.
- **The cutover crashed the old host.** During the cable moves, the old host's 10 Gb card logged a
  fatal PCIe bus error and the kernel panicked. A hard power cycle recovered it. The same slot had
  logged an identical error months earlier, on a day the chassis was opened. Pulling the WAN module
  logged a clean link-down, so the likelier cause is card seating or riser, not the hot-pull
  itself. That card was going to carry the host's management bridge. That move now waits until the
  card is reseated and has run clean.
- **The runbook had a gap.** It unplugged the old host's only LAN link before the new router was
  up. That cut off everything else on that host, including the Wi-Fi controller, for the rest of
  the cutover. Next time, move the host to a second, pre-tested switch port, and only then pull
  the firewall's link.
- From shutting down the VM to the desktop being back online took about 75 minutes, against a
  planned 30. That included the host crash and a cable modem that ignored DHCP until it was
  power-cycled.

## Alternatives considered

**Keep the firewall as a VM.** Zero cost, already working. Rejected: it keeps every cluster
operation coupled to the network, and it makes the cluster's own failure test destructive.

**A smaller, older 1U server (R210 II class).** Cheaper, but 1 GbE onboard and a 2011 platform.
It would need an add-in NIC anyway and halve the WAN without one.

**Move the existing 10 Gb card from the cluster node into the new box.** Saves about $20.
Rejected: it's a Broadcom card, and its FreeBSD driver is the less-trodden path compared with
Intel's. The cluster node needs that card for its own 10 Gb uplink. Leaving it in place also kept
rollback to a cable move.

**The same server bundled with a low-power Xeon and a spare CPU.** Rejected on cost. Shipping
alone nearly doubled the price, and a Xeon can be added to the cheaper box later if needed.

---

## The generalization

The quorum witness made the cluster *survive* losing a node. It did nothing about the fact that
losing one specific node also took down the network needed to notice, diagnose and fix it.
**Infrastructure that is needed to repair a system shouldn't run on that system.** The firewall
was the most important case. The password vault, which runs on the same cluster and holds the
credentials needed in an outage, is the next one.
