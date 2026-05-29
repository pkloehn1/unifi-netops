# Debian/Ubuntu host networking reference

Tooling and config patterns for Linux hosts on a UniFi network, including
multi-NIC, VLAN, bonding, macvlan, and Docker Swarm.

## Tooling

- **iproute2**: `ip -br link`, `ip -br addr`, `ip -d link show type bond|bridge|vlan`,
  `bridge link`, `ip -ts monitor link` (watch carrier events live).
- **ethtool**: `ethtool <if>` (speed/duplex/link), `ethtool --show-eee <if>`,
  `ethtool -S <if>` (RX/TX errors, CRC), `ethtool -i <if>` (driver).
- **lldpd / lldpcli**: `lldpcli show neighbors` — authoritative NIC → switch +
  port mapping. Install on every host; it's the fastest way to learn the L1
  topology without guessing.
- **netplan** (`/etc/netplan/*.yaml`) or **systemd-networkd** (`networkctl
  status`) for config; `nmcli` on NetworkManager hosts.
- Reachability/capture: `ping`, `arping`, `mtr`, `iperf3`, `tcpdump`, `ss`.

## Multi-NIC / ARP flux

- Two NICs on the **same subnet** with default `arp_filter=0` make the host
  answer ARP for its IPs out *both* NICs — the switch then sees the MAC move
  between ports. Mitigate:
  ```
  net.ipv4.conf.all.arp_filter = 1
  net.ipv4.conf.all.arp_ignore  = 1
  net.ipv4.conf.all.arp_announce = 2
  ```
  Plus policy routing (separate `rt_tables` + `ip rule`) so each NIC's replies
  egress its own interface. Better still: don't put two NICs on one subnet
  unless they're a single bond.

## Bonding (LACP)

- 802.3ad LACP requires all members on the **same switch** (or a real MLAG
  pair). Confirm the partner with `ad_partner_mac` in
  `ip -d link show <bond>` — it should be one switch.
- A bond running on a **single member** (others down) is degraded; restore the
  second leg or fix the definition.
- Do **not** build a bond with members on two non-stacked switches — that
  presents one MAC on both switches and trips loop/MAC-move protection.

## VLANs

- 802.1q sub-interfaces (`enpXsY.40`) or netplan `vlans:`. Make sure a VLAN
  reaches a host via **one** switch path; spanning the same VLAN to NICs on two
  switches through the host is a loop.

## macvlan + Docker Swarm

- **Overlay/VXLAN** ports: 2377/tcp (swarm mgmt), 7946/tcp+udp (gossip),
  4789/udp (overlay data).
- **macvlan** dual-homing: a config-only network (the IPAM template, with
  `--subnet/--gateway/--ip-range/-o parent=<nic>`) plus a swarm-scope macvlan
  created `--config-from` it. Create swarm-scope macvlan from a **manager that
  owns the parent NIC** (temp-promote the node if needed; poll
  `docker info -f '{{.Swarm.ControlAvailable}}'` after promote before creating).
- **IPAM is fragile**: current Docker renders `IPAM.Config: null/[]` for
  macvlan even when working — verify by attaching a one-shot probe service and
  reading the assigned IP, *not* by `docker network inspect`.
- `--ip-range x.x.x.x/32` = exactly one VIP. Orphaned `Created`/`Exited`
  containers hold that IP and block new tasks with "no available IPv4
  addresses" — remove them before recreating.
- Promotion/`docker node` and `docker network rm` are mutations — get operator
  approval.

## Docs

- netplan: https://netplan.io
- systemd-networkd: `man systemd.network`
- Linux bonding: https://www.kernel.org/doc/html/latest/networking/bonding.html
- Docker networking: https://docs.docker.com/network/ (overlay, macvlan, swarm)
