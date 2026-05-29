# Fabric flap / loop diagnostic playbook

A repeatable procedure for "connectivity / SSH / RDP flaps fleet-wide on a
UniFi network." Goal: separate Wi-Fi from wired, pin the failing element, and
prove it by bisection — not by guessing.

## 1. Reachability monitor (from a WIRED host)

Run from a host wired into the fabric (not Wi-Fi). Sampling ICMP + a TCP service
to both a suspect host *and* the gateway distinguishes a single-host fault from
a fabric-wide one, and timestamps the outage windows.

**Bash:**
```bash
TARGET=10.0.40.31; GW=10.0.40.1; LOG=flap.log
while true; do
  ts=$(date -Is)
  t=$(ping -c1 -W1 "$TARGET" 2>/dev/null | sed -n 's/.*time=\([0-9.]*\) ms/\1/p'); t=${t:-DOWN}
  g=$(ping -c1 -W1 "$GW" 2>/dev/null | sed -n 's/.*time=\([0-9.]*\) ms/\1/p'); g=${g:-DOWN}
  s=$(timeout 2 bash -c "exec 3<>/dev/tcp/$TARGET/22" 2>/dev/null && echo ok || echo FAIL)
  printf '%s tgt=%s gw=%s ssh22=%s\n' "$ts" "$t" "$g" "$s" | tee -a "$LOG"
  sleep 2
done
```

**PowerShell (Windows host):**
```powershell
$t="10.0.40.31"; $gw="10.0.40.1"
while ($true) {
  $ts=(Get-Date).ToString("o")
  $pt=(Test-Connection $t  -Count 1 -Quiet)
  $pg=(Test-Connection $gw -Count 1 -Quiet)
  $rdp=(Test-NetConnection $t -Port 3389 -WarningAction SilentlyContinue).TcpTestSucceeded
  "$ts tgt=$pt gw=$pg rdp=$rdp" | Tee-Object -FilePath flap.log -Append
  Start-Sleep 2
}
```

Reading it: if a host's own NIC shows **no carrier loss** (`ip -ts monitor
link` / `Get-NetAdapter`) yet it still can't reach peers, the fault is
**upstream in the fabric**, not local. Both target and gateway dropping
*together* points at a common upstream element (an inter-switch link / a
switch), not the target host.

## 2. Capture controller/switch syslog

Point the UniFi controller's remote syslog at a collector (UDP 514). A minimal
collector that timestamps each datagram:

```python
import socket, datetime
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.bind(("0.0.0.0", 514))
with open("udm-syslog.log", "a", buffering=1) as f:
    while True:
        data, addr = s.recvfrom(65535)
        line = f"{datetime.datetime.now().astimezone().isoformat()} src={addr[0]} {data.decode('utf-8','replace').rstrip()}\n"
        f.write(line); print(line, end="")
```
(Bind 514 needs privilege; a container publishing `514/udp` avoids host root.)

Grep for the high-signal events:
- `Port N moving … to Forwarding` — STP transitions (which port).
- `entering isolated state` — a device lost its uplink.
- `inform … Timeout` / `inform failed` — device lost the controller.
- CEF `Wired/WiFi Client Connected/Disconnected` — port, VLAN, link speed per MAC.

## 3. Correlate

Line up the monitor's outage windows against the syslog timestamps. If STP
churn is confined to **one port** and lands on the same cadence as the outages,
that link/segment is the locus. A precise ~60s cadence ⇒ a periodic
renegotiation (marginal/mismatched SFP, or loop-protection auto-recovery), not
a random cable.

## 4. Bisect physically

Remove one variable at a time and re-measure against the monitor + the STP
transition count:
- Pull one dual-homed host's second-switch leg → does the churn stop?
- **Account for every host**, including ones that look single-homed until you
  check (`ad_partner_mac` on a bond, an AP/PDU uplink). A common mistake is
  declaring "it's the link" after testing only some hosts.
- If churn persists with **all** host cross-switch legs removed, the redundant
  path isn't via a host — it's the inter-switch link itself (transceiver/cable/
  port/speed). See `references/unifi.md` SFP section.

## 5. Confirm the fix

After a change, watch for ~10 minutes (well beyond the old cadence):
- monitor log stays clean (no DOWN/FAIL), and
- the STP transition count for the suspect port stops increasing.

Don't declare "fixed" on a window shorter than several multiples of the
original flap interval.

## macvlan / Swarm edge restore (when the edge node won't get its VIP)

- Pre-flight: refuse to recreate while the consuming service still exists (it
  re-grabs the single `/32`). Stop the stack first.
- Clear orphaned `Created`/`Exited` containers holding the `/32` before
  recreating, or new tasks fail with "no available IPv4 addresses."
- Recreate from a manager that owns the parent NIC; after `docker node
  promote`, poll `docker info -f '{{.Swarm.ControlAvailable}}'` before creating
  swarm-scope networks.
- Verify by **probe**, not inspect: attach a one-shot service to each macvlan
  and confirm it gets the expected VIP. Demote the node afterward (use an EXIT
  trap so it fires even on failure).
