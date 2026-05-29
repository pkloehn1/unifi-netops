# UniFi fabric reference

UniFi-specific knowledge for diagnosis and config review. Vendor-neutral
examples; substitute your own addresses/ports.

## Model

- **Gateway / controller** (UDM / UDM-Pro / UDM-SE / Cloud Gateway / self-hosted
  Network app): routing, firewall, DPI/IDS-IPS, DHCP, and the controller that
  adopts/provisions switches and APs.
- **USW switches**: access (1G + PoE) and aggregation (SFP+/10G). Inter-switch
  links ride SFP/SFP+ uplinks.
- **APs (U6/U7)**: wired uplink to a switch; client roaming generates a lot of
  benign assoc/disassoc syslog — don't mistake it for fabric instability.

## Spanning tree, loops, and inter-switch links

- UniFi runs **RSTP** by default; switches are **not MLAG/stacked**. A single
  uplink between two switches is one STP segment and a single point of failure.
- `kernel: Port N moving from Blocking/Learning to Forwarding` repeated on **one
  port** = that link is flapping (link-down events may not be forwarded at the
  syslog level) or STP keeps cutting/clearing a loop there.
- **Loop signature**: a host dual-homed to two switches on the same VLAN (with
  host forwarding / ARP flux) is a redundant path the fabric blocks by cycling
  the inter-switch link. Symptoms downstream: PoE/PDU "entering isolated state",
  switch "inform timeout" to the controller, wired clients disconnecting across
  VLANs — all *co-victims* of the one cycling segment.
- **Loop protection** can err-disable a port and auto-recover on a timer
  (~minute), producing a metronomic flap.

## SFP / SFP+ optics

- **SFP+ (10G) port ↔ SFP (1G) port** flaps unless the SFP+ side is pinned to a
  fixed **1 Gbps** and both ends use matched optics (or a DAC rated for the
  speed). UniFi exposes per-port **Link Speed / operation** for this.
- **EEE / Green Ethernet is not a UniFi UI setting** and is unsupported on SFP.
  For a flapping SFP/SFP+ link the real levers are: reseat → replace the
  transceiver/DAC/cable, move to a different port on each switch, pin speed,
  and check port error counters. RJ45/copper SFP modules are a common flap
  source (heat + a PHY whose EEE can't be disabled).

## Integration API + diagnostics

- **Official UniFi API (X-API-KEY)**: enable at the controller. Local Network
  Integration API is per-controller-version — its docs live under
  **UniFi Network → Integrations**; keys at unifi.ui.com → Settings → API Keys.
- Useful read endpoints/tools: list devices (gateway/switch/AP), get device by
  MAC (port table, stats), list/search active clients (which port/VLAN a MAC is
  on, and whether it moves), network topology, list VLANs.
- **Local-mode gotcha**: connect by hostname for TLS cert match but pin the
  resolution to the controller's LAN IP (curl `--resolve`, Docker `--add-host`)
  so you reach the local controller, not a hairpin to a cloud/proxy edge.
- **Remote syslog**: point the controller's syslog server at a collector
  (UDP 514). UniFi emits CEF client connect/disconnect events (with port, VLAN,
  link speed per MAC), STP transitions, and device provision/inform events —
  the richest signal for correlating fabric flaps.
- SSH to the gateway/switches exposes lower-level tools (`info`, `swctrl`,
  `mca-cli`) when the API is insufficient.

## Documentation & support

- UniFi help center: https://help.ui.com
- UniFi developer portal (Integration / Site Manager API): https://developer.ui.com
- UniFi community forums: https://community.ui.com (search switch STP / SFP /
  autoneg / loop-protection threads)
- Version-local API docs: UniFi Network → Integrations
