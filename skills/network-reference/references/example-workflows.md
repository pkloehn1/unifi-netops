# Example diagnostic workflows

Real-shape case studies, generalised. Each follows the same arc — **symptoms
→ bisection → root cause → fix now → durable mitigation** — because the
bisection moves are what transfer across incidents; root causes vary.

When you face a new puzzle, scan for a case whose *symptoms* rhyme with
yours and try its bisection move first.

---

## 1. Fleet-wide latency from an SFP+↔SFP inter-switch link flap

### Symptoms

- SSH/RDP sessions across **multiple** wired hosts hang and recover on a
  rough ~minute cadence.
- Wi-Fi clients are affected too — looks like "the whole network" is
  unstable, no single device at fault.
- Rebooting hosts changes nothing; the cadence is the same.

### Bisection

1. Run a **wired reachability monitor** (see
   [diagnostics-playbook.md](diagnostics-playbook.md)) from one wired host
   to another that crosses the suspected inter-switch link. Pin outage
   windows to absolute timestamps.
2. Send the gateway/switches' syslog (UDP 514) to a collector. Watch for
   `STP: Port N moving … to Forwarding` repeating on the **same port** and
   for inter-switch topology-change events.
3. Correlate by timestamp. If STP churn is confined to one port and every
   blackout aligns with that port's Forwarding ↔ Blocking transitions, the
   segment under that port is the fault.

### Root cause (real example)

- The aggregation-switch's SFP+ port was acting as a 10G-to-1G uplink to a
  USW-24-PoE-style access switch terminated in a 1G SFP module. The link
  was negotiating, dropping, renegotiating on a sub-minute cadence.
- Reseating the SFP+ transceiver stopped the flap immediately. A clean
  10-minute monitor confirmed.

### Why this is mistaken for DNS / Wi-Fi / hosts

UniFi switches **are not MLAG/stacked by default**, so an inter-switch
uplink is a single STP segment and a single point of failure. Every
cross-switch path — wired and Wi-Fi — traverses it, so its flaps show up
as "everything is broken."

### Fix now

Reseat the optic/cable and verify with `swctl portstat` or
`get_device_details` that flap counters have stopped advancing.

### Durable mitigation

- **Pin the SFP+ port to 1 Gbps / Full Duplex** when the other side is a
  1G SFP. Auto-negotiation across a rate boundary is the most reliable
  trigger.
- **Use matched optics** (same vendor both sides) or a DAC if the run is
  short enough. Eliminates "vendor compatibility" as a variable.
- Add a port-stats poller (UniFi MCP `get_device_details` or `swctl
  portstat`) to your rotation — flap counts trend before they break.

### Invariants this reinforces

- "UniFi switches are not MLAG by default" — every ISL is a SPOF.
- "SFP+↔SFP flaps unless pinned" + "EEE is not the lever" — don't chase an
  EEE toggle for SFP/SFP+ link flaps; it isn't configurable in the UI and
  is unsupported on SFP.

---

## 2. macvlan IPAM looks wiped after a Docker daemon restart

### Symptoms

- After a Docker daemon restart on the macvlan-host node, a
  macvlan-attached service is stuck `0/1`.
- Tasks fail with: `no available IPv4 addresses on this network's address
  pools`.
- `docker network inspect <macvlan>` shows `IPAM.Config: null`
  (config-only) or `IPAM.Config: []` (swarm-scope). Looks like the IPAM was
  erased.

### Bisection

**Don't trust `docker network inspect` here.** On current Docker, a
working macvlan often renders its `IPAM.Config` as `null/[]` — it's a
render artifact, not a real wipe. The only reliable test is to attach a
probe:

```bash
docker service create --name probe --network <macvlan> \
  --constraint node.hostname==<macvlan-host> --replicas 1 \
  --restart-condition none alpine sleep 60
docker service ps probe --no-trunc
```

If the probe gets an IP, the network is healthy and the missing IPAM is
just the render — look elsewhere (replicas, image, placement). If the
probe **also** fails with "no available IPv4 addresses", the IPAM has
actually been wiped (or all IPs in the `--ip-range` are held).

### Common false positive: orphan `Created` containers

A previous failed attach can leave containers in `Created` state holding
the only `/32` IP from your `--ip-range`:

```bash
docker ps -a --filter network=<macvlan> --format '{{.ID}} {{.Status}}'
docker rm <orphan-id>          # releases the IP
```

The next attempt then gets the IP. Always sweep these before concluding
IPAM is wiped.

### Root cause (real example)

The restart did wipe IPAM on both the config-only and swarm-scope macvlan
networks. Recovery:

1. From a manager: `docker network rm <swarm-scope macvlan>`.
2. SSH to the macvlan-host (the node that owns the parent NIC); promote it
   to manager so it can author a swarm-scope network from `--config-from`;
   `docker network rm <config-only network>`.
3. Re-run the idempotent create script on the macvlan-host. It must
   create the config-only network first, then the swarm-scope one with
   `--config-from <config-only>`.
4. Demote the macvlan-host back to worker.
5. Redeploy the stack.

### Durable mitigation

- Keep the create script in-repo, idempotent, and runnable only from the
  node that owns the parent NIC.
- Bake the **probe-service verification step** into your runbook
  explicitly so future-you doesn't fall for the inspect render.
- Treat `inspect` IPAM output as advisory; treat probe-service IP as
  authoritative.

### Invariants this reinforces

- "macvlan IPAM looks empty even when healthy" — inspect lies; verify with
  a probe.
- swarm-scope macvlans must be authored on the node that owns the parent
  NIC (so that node must be a manager during creation).

---

## 3. SwarmKit's compose loader rejects per-attachment `mac_address`

### Symptoms

- `docker stack deploy -c stack.yml <name>` rejects the file with:
  `services.<svc>.networks.<net> Additional property mac_address is not
  allowed`.
- `docker compose config -c stack.yml` (standalone loader) accepts the
  **same** file cleanly. CI didn't catch it because CI uses
  `docker compose config`.
- The compose was committed by a PR whose stated goal was to pin
  per-container MAC addresses for upstream-DHCP reservations.

### Bisection

The two loaders are not the same:

| Loader | Used by | Strictness |
|---|---|---|
| Standalone compose loader | `docker compose ...`, `docker compose config` | Permissive |
| SwarmKit compose loader | `docker stack deploy`, `docker stack config` | Strict — rejects per-attachment `mac_address` and some other keys |

Validate every commit that touches a stack file with:

```bash
docker stack config -c stack.yml >/dev/null
```

That is the loader production sees, and it fails when `docker compose
config` would pass.

### Root cause

Per-attachment `mac_address` keys under
`services.<svc>.networks.<net>.mac_address` are not supported by the
SwarmKit compose loader.

### Fix now

Revert the `networks:` block to **list form**:

```yaml
networks:
  - frontend
  - dmz_macvlan
  - lan_macvlan
```

The original goal (stable upstream-DHCP reservation for the front-door
service) is preserved by pinning **IP** instead of MAC: set the macvlan's
`--ip-range a.b.c.d/32` and reserve **by IP** on the upstream gateway. The
service gets the same IP on every recreate; the reservation is stable.

### Durable mitigation

- Add a CI step that runs `docker stack config -c stacks/*/docker-compose.yml
  >/dev/null` against every stack file. `docker compose config` would not
  have caught this.
- If a use case genuinely requires per-container MAC pinning under swarm,
  use a deploy-time strip (Python + PyYAML pops `mac_address` from every
  attachment before piping into `docker stack deploy -c -`). Document this
  as a workaround, not a pattern.

### Invariants this reinforces

- Standalone-compose ≠ SwarmKit-compose. Validate with the loader you
  actually deploy with.
- DHCP reservations don't require MAC pinning — IP pinning via
  `--ip-range /32` is enough when the upstream reserves by IP.

---

## 4. "It's always DNS" — wildcard AAAA + in-container TLS verification

### Symptoms

- A containerized client (e.g. a MCP server, a CI runner, a webhook
  sender) intermittently fails to reach `https://service.example.lan`
  with TLS errors or `connection reset` / `connection refused`.
- The **same hostname works from the host shell**.
- The cert is valid (chain, not-after, SAN includes the hostname) and the
  target service is reachable on the LAN from other clients.
- DNS *appears* to resolve, but the container hits a different IP than
  the host does — or returns an AAAA the container can't reach.

### Bisection

Resolve from inside the container vs. outside:

```bash
# Host
dig +short service.example.lan A AAAA

# Inside the container the client is actually running in
docker exec <container> sh -c \
  'getent ahosts service.example.lan'
```

Compare A and AAAA. If the container's resolver returns an AAAA for
`*.example.lan` that points to an unreachable host, TLS handshakes hang
because most clients prefer AAAA per RFC 6724.

Confirm with an explicit TLS handshake from inside the container:

```bash
docker exec <container> sh -c \
  'openssl s_client -servername service.example.lan \
     -connect service.example.lan:443 -brief </dev/null'
```

If `-4` (force IPv4) succeeds and `-6` fails, the AAAA is the lie.

### Root cause (real example)

A `*.example.lan AAAA <something>` wildcard returned IPv6 records for
every hostname under the zone, including ones that only had functional
IPv4 reachability. Clients that prefer AAAA tried the broken IPv6 path
first; timeouts looked like the application or TLS was at fault.

Independently, in-container resolution didn't return the same IPv4 the
host saw, because the operator was using split-horizon: the controller's
hostname needed to resolve to the **internal** IP for hosts on the LAN,
but the container's resolver chain went out to a public path.

### Fix now

In-container hostname **pin** via `--add-host`:

```bash
docker run --add-host service.example.lan:10.0.10.1 ... <image>
```

This makes `getaddrinfo()` return the internal IPv4 directly, bypassing
both the broken AAAA and any wrong-IP resolution. TLS verification still
passes — verification is over the **cert SAN vs. SNI hostname**, not the
IP, so pinning the IP while keeping the hostname intact preserves it.

### Durable fix

- **Remove or null the AAAA wildcard.** A `*.example.lan AAAA …` is rarely
  what you want; replace it with an SOA-style "no AAAA exists" answer so
  clients fall back to A immediately instead of stalling on a broken IPv6.
- **Use split-horizon DNS on the gateway** (the UDM's "Local DNS" /
  controller-side overrides). Internal hostnames resolve to internal IPs
  for clients on the LAN, external resolvers continue to see whatever
  public path is appropriate. Containers that query the LAN gateway then
  see the same answer the host does.
- For containers that don't (can't) query the LAN gateway, `--add-host`
  becomes a documented workaround, not the fix.
- Whenever a containerised client connects to an internal HTTPS endpoint
  by name, document the resolver path it uses (Docker's default DNS, host
  networking, explicit `--dns`, custom `resolv.conf`) so the answer is
  reasoned about ahead of time.

### Invariants this reinforces

- A working hostname + a valid cert + a broken AAAA = an application that
  appears to fail "on the network." It's not the network — it's DNS.
- TLS verification is over the **SAN-vs-SNI hostname**, not the IP.
  Pinning the IP (`--add-host`) while keeping the hostname intact
  preserves TLS verification.
- Split-horizon is a DNS-server feature; containerised clients only
  benefit from it if they query the DNS server that knows.

---

## 5. A network-facing Swarm daemon parks at `0/1`, blamed on an upstream-API auth error

### Symptoms

- A long-running network service deployed via `docker stack` (e.g. a
  dynamic-DNS updater keeping the WAN `A` record current) sits at `0/1` for
  days; the network function it provides silently stops.
- Its logs prominently show an **upstream-API auth warning** — favonia
  cloudflare-ddns logs `The Cloudflare API token appears to be invalid:
  Invalid API Token (1000)` — so suspicion falls on a bad/rotated token.
- `docker service ps <svc>` shows the task repeatedly reaching `Complete`
  (exit 0) within seconds, then the service gives up and stays at `0/1`.
- Re-entering or rotating the token changes nothing.

### Bisection

1. **Pull the full container log, not a filtered tail.** Grep for the
   lifecycle markers, not just the scary auth line:
   ```bash
   ssh <swarm-node> 'docker logs <container-id> 2>&1 \
     | grep -E "Caught signal|Bye|already up to date|Invalid"'
   ```
   If the daemon logs a *successful data-plane op* (`already up to date`)
   **and** `Caught signal: terminated` → `Bye`, the exit is an **external
   SIGTERM**, not a crash and not an auth failure.
2. **Separate cosmetic from causal.** favonia validates the token via
   Cloudflare's **User-scope** `/user/tokens/verify` endpoint, which returns
   1000 for a **Zone-scoped** token even though its `DNS:Edit` permission
   works. The `already up to date` line proves the token is functional — the
   1000 warning is a red herring. (Generalises: an alarming upstream-API line
   is not the cause if the data-plane op it gates is still succeeding.)
3. **Find who sends the SIGTERM.** With no concurrent `docker stack deploy`
   running, the sender is Swarm's own update orchestrator. Inspect the service:
   ```bash
   docker service inspect <svc> \
     --format '{{json .Spec.UpdateConfig}}{{"\n"}}{{json .Spec.TaskTemplate.RestartPolicy}}'
   ```
   Compare `UpdateConfig.Monitor` (default `5000000000` ns = **5s**) against
   the healthcheck's `StartPeriod`. Monitor < StartPeriod is the smoking gun.

### Root cause (real example)

A compound Swarm-lifecycle bug, mistaken for an auth failure:

- `update_config.monitor` was the **5s default**; the service's healthcheck had
  `start_period: 10s`. The orchestrator watched each new task for only 5s, saw
  it still "starting", declared the update **failed**, rolled it back, and
  **SIGTERM'd the task at ~30s**.
- `restart_policy.condition: on-failure` then **refused to restart** the clean
  exit-0. For a daemon, an external-SIGTERM exit-0 *must* restart; `on-failure`
  reads it as "job finished" and **parks the service at `0/1`**.
- `max_attempts: 3` / `window: 120s` compounded it — three quick exits
  exhausted attempts and hit the [moby#43712](https://github.com/moby/moby/issues/43712)
  parking trap.

### Fix now

```yaml
deploy:
  restart_policy:
    condition: any          # not on-failure: a daemon's clean SIGTERM exit must restart
    max_attempts: 0         # unlimited — never exhaust into the moby#43712 park
    delay: 30s
    window: 300s
  update_config:
    monitor: 30s            # must exceed any healthcheck start_period
    order: stop-first       # singleton — never two updaters racing one record
# and DROP the no-op `healthcheck: ["CMD", "true"]` whose start_period raced monitor
```

### Durable mitigation

- For any **long-running daemon** under Swarm, default to
  `restart_policy.condition: any` — `on-failure` is for run-to-completion
  jobs, and a daemon that takes an external SIGTERM exits 0.
- Set `update_config.monitor` **explicitly** and keep it `>` the largest
  healthcheck `start_period` in the service. Don't rely on the 5s default.
- Don't add a healthcheck just to "have one." A `["CMD", "true"]` probe does
  nothing but contribute a `start_period` that can race the monitor window.
- When an upstream-API warning appears in a daemon's logs, confirm the
  **data-plane op** (the thing the service actually does) before chasing the
  warning — favonia's `already up to date` is the proof the token works.

### Invariants this reinforces

- A long-running Swarm daemon that exits 0 on an external SIGTERM parks at
  `0/1` unless `restart_policy.condition: any`.
- `update_config.monitor` must exceed the healthcheck `start_period`, or the
  update orchestrator rolls back and SIGTERMs the new task mid-startup.
- The alarming log line is not the cause if the data-plane op it gates still
  succeeds — verify the op, not the warning.

---

## How to use these

When you face a new diagnostic puzzle, scan for the case whose
**symptoms** rhyme with yours and run its bisection move first. The
bisection moves transfer; root causes vary. When you finish a real
diagnosis, consider adding it here.
