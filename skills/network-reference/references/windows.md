# Windows 11 host networking reference

PowerShell tooling and patterns for Windows 11 hosts on a UniFi network. Run an
elevated PowerShell for most of these.

## Inventory & link state

- `Get-NetAdapter` — interface list, link state, speed.
- `Get-NetAdapter | Format-List Name,InterfaceDescription,LinkSpeed,MediaConnectionState,FullDuplex`
- `Get-NetIPConfiguration` / `Get-NetIPAddress` — IPs, gateways, DNS per interface.
- `Get-NetAdapterStatistics` — counters; look for receive/transmit errors and
  discards (the Windows analogue of `ethtool -S`).

## NIC advanced properties (EEE / offloads / speed)

- `Get-NetAdapterAdvancedProperty -Name <if>` — lists vendor keywords incl.
  **Energy-Efficient Ethernet (EEE)**, Green Ethernet, Flow Control, Jumbo,
  and offloads.
- Pin speed / disable EEE on a flapping link (host side):
  ```powershell
  Set-NetAdapterAdvancedProperty -Name <if> -DisplayName "Energy Efficient Ethernet" -DisplayValue "Disabled"
  Set-NetAdapterAdvancedProperty -Name <if> -DisplayName "Speed & Duplex" -DisplayValue "1.0 Gbps Full Duplex"
  ```
  (Display names vary by driver; enumerate first with
  `Get-NetAdapterAdvancedProperty`.) Note: the UniFi *switch* side has no EEE
  toggle — host-side is the only place you can disable it.
- `Get-NetAdapterEee -Name <if>` (where supported) shows negotiated EEE state.

## Teaming / VLAN

- **LBFO / SET teaming**: `Get-NetLbfoTeam`, `Get-NetLbfoTeamMember`,
  `Get-NetLbfoTeamNic`. As with Linux bonding, team members must land on **one
  switch** unless it's a real MLAG pair. LACP teams: `-TeamingMode Lacp`.
- VLAN tagging is driver/teaming-dependent: `Set-NetLbfoTeamNic -VlanID <id>`
  or the NIC's advanced "VLAN ID" property. Keep a given VLAN on one switch
  path per host (same loop rule as Linux).

## Reachability & capture

- `Test-NetConnection <host> -Port <p>` — TCP reachability + traceroute-ish path.
- `Test-NetConnection <host> -DiagnoseRouting` / `-InformationLevel Detailed`.
- `Get-NetNeighbor` — ARP/ND cache (spot MAC/IP anomalies).
- **pktmon** (built-in capture): `pktmon start --capture --pkt-size 0 -f cap.etl`
  then `pktmon stop`; convert with `pktmon etl2pcap cap.etl`. Use for
  flap/loss correlation when you can't install Wireshark.
- `arp -a`, `ipconfig /all`, `route print` for quick checks.

## Weak-host / multi-NIC

- Windows uses the **strong host model** by default (an interface won't answer
  for IPs on another interface) — the analogue of Linux ARP-flux mitigation.
  Keep `Get-NetIPInterface -AddressFamily IPv4 | Select ifIndex,WeakHostSend,WeakHostReceive`
  at the default (disabled). Prefer one interface per subnet.

## Docs

- Windows networking cmdlets: https://learn.microsoft.com/powershell/module/netadapter/
- pktmon: https://learn.microsoft.com/windows-server/networking/technologies/pktmon/pktmon
- NIC teaming (LBFO/SET): https://learn.microsoft.com/windows-server/networking/technologies/nic-teaming/nic-teaming
