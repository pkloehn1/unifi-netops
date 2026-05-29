---
name: network-reference
description: >-
  Reference knowledge and a diagnostic playbook for UniFi networks operated
  with Debian/Ubuntu and Windows 11 hosts. Covers the UniFi fabric model
  (gateways, USW switches, STP/loops, SFP/SFP+ optics, the Integration API),
  per-OS network tooling, Docker Swarm overlay/macvlan, and a fabric
  flap/loop diagnostic procedure. Use when diagnosing or reviewing any UniFi,
  Linux, Windows, or container networking — fabric instability, link flaps,
  loops, VLAN/bonding/macvlan config, or connectivity that drops fleet-wide.
license: MIT
compatibility: >-
  Vendor-neutral knowledge. Examples assume Bash + iproute2/ethtool/lldpd on
  Debian/Ubuntu and PowerShell on Windows 11. A UniFi Network Integration API
  key (and optionally a UniFi MCP server) enables live controller queries but
  is not required.
metadata:
  author: pkloehn1
  version: "0.1.0"
allowed-tools: Read Bash(ip:*) Bash(ethtool:*) Bash(lldpcli:*) Bash(dig:*) Bash(ping:*) Bash(ssh:*) Bash(docker:*) WebFetch WebSearch
---

# UniFi + Debian/Ubuntu/Windows 11 Network Reference

Reference for diagnosing and configuring UniFi networks alongside Linux and
Windows hosts. This file is the index; load the focused reference files only
when a task needs them (progressive disclosure).

- UniFi gateway/switch/AP model, STP/loops, SFP optics, Integration API, syslog
  → [references/unifi.md](references/unifi.md)
- Debian/Ubuntu host networking (netplan, bonding, VLAN, macvlan, ARP, Docker
  Swarm) → [references/linux.md](references/linux.md)
- Windows 11 host networking (PowerShell, NIC advanced properties, LBFO
  teaming, VLAN) → [references/windows.md](references/windows.md)
- Fabric flap/loop diagnostic procedure (monitor + syslog + correlate + bisect)
  → [references/diagnostics-playbook.md](references/diagnostics-playbook.md)

## Operating principles

1. **Verify, don't assume.** Tie every claim to a command output, log line, or
   doc. State uncertainty explicitly.
2. **Diagnose before prescribing.** Observe/reproduce first; correlate a
   reachability monitor against controller/switch syslog by timestamp.
3. **Bisect.** Remove one variable at a time and account for *every* candidate
   device/link, not just the obvious one.
4. **Mutations need operator approval** (controller/switch/DNS/host changes,
   `docker network rm`, port/power cycling).

## Universal invariants

These hold regardless of platform and are the ones most often missed:

- **UniFi switches are not MLAG/stacked by default.** An inter-switch uplink is
  a single STP segment and a single point of failure. Repeated
  `Port N moving … to Forwarding` on one port = that link/segment is flapping
  or a loop is being cut there.
- **Never bridge a VLAN across two switches through a host.** One NIC per
  switch on the same VLAN (with host forwarding or ARP flux) is a loop the
  fabric breaks by cycling the inter-switch link. Single-home each VLAN per
  host, or LAG to a single switch.
- **Multi-NIC hosts on a shared subnet need ARP-flux mitigation** or they
  advertise a MAC out multiple ports and trigger MAC-move/loop-protection.
- **SFP+ (10G) ↔ SFP (1G) flaps** unless the SFP+ side is pinned to 1 Gbps with
  matched optics. A precise ~minute flap cadence usually means renegotiation or
  a marginal/mismatched module — not a random cable fault.
- **EEE / Green Ethernet is not configurable in the UniFi UI** and is
  unsupported on SFP. Don't chase an EEE toggle for SFP/SFP+ flaps; the levers
  are the transceiver/DAC/cable, the port, and fixed speed.
- **macvlan IPAM looks empty even when healthy.** `docker network inspect`
  shows `IPAM.Config: null/[]` on current Docker for working macvlan networks —
  verify with a probe service's assigned IP, never by reading that field.

## Quick triage entry point

Fleet-wide connectivity flapping (SSH/RDP dropping on multiple hosts):

1. Run a **wired reachability monitor** from a wired host (it isolates Wi-Fi
   from fabric and pins outage windows).
2. Capture **controller/switch syslog** (UDP 514) and look for STP transitions,
   uplink "isolated state", inform timeouts, and client connect/disconnect.
3. **Correlate** the two by timestamp; STP churn confined to one port names the
   link/segment.
4. **Bisect** physically (pull one dual-homed host/link at a time).

Full procedure + copy-paste monitors (Bash + PowerShell):
[references/diagnostics-playbook.md](references/diagnostics-playbook.md).
