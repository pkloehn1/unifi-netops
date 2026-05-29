# unifi-netops

A Claude Code plugin that adds a **principal network-engineer subagent** and a
**reference skill** for **UniFi** networks operated alongside **Debian/Ubuntu**
and **Windows 11** hosts.

It is built for network *diagnosis and config review*: fabric/STP/link-flap and
loop triage, UniFi gateway/switch config review, host NIC/VLAN/bonding/macvlan
issues on Linux or Windows, Docker Swarm overlay/macvlan networking, and
SFP/SFP+ transceiver faults. The subagent works in its own context and returns
a summary, so log-heavy diagnosis doesn't flood your main session.

## What's inside

| Component | Name | Purpose |
|---|---|---|
| Subagent | `network-engineer` | Diagnostic persona with read-only-by-default tools; preloads the reference skill |
| Skill | `network-reference` | Vendor-neutral UniFi/Linux/Windows reference + a flap/loop diagnostic playbook (progressive disclosure via `references/`) |

The skill conforms to the open [Agent Skills](https://agentskills.io)
specification, so it is portable to other skills-compatible agents.

## Install

```text
/plugin marketplace add pkloehn1/unifi-netops
/plugin install unifi-netops
```

Or test locally without installing:

```bash
claude --plugin-dir /path/to/unifi-netops
```

Then delegate to it (e.g. "use the network-engineer agent to triage the
switch flapping") or open the reference with `/unifi-netops:network-reference`.

## Prerequisites

- **Optional but recommended: a UniFi Network Integration API key** for live
  controller queries.
  - **Tested MCP server: [`enuno/unifi-mcp-server`](https://github.com/enuno/unifi-mcp-server)**
    (`ghcr.io/enuno/unifi-mcp-server`). This is the **only** UniFi MCP this
    plugin has been tested with — the agent's controller-query guidance assumes
    its tool surface (e.g. `list_devices_by_type`, `get_device_by_mac`,
    `list_active_clients`, `get_network_topology`, `search_clients`,
    `list_vlans`). Other UniFi MCP servers may work but are unverified; adjust
    expectations accordingly.
  - Configure that MCP at the **project/user level** —
    **plugin subagents cannot ship `mcpServers`**, so the connection must
    already exist in the session. Keep the API key in environment-local config
    (e.g. a gitignored `.env`), never in this repo.
  - The agent is still useful with **just Bash/Read/Web** for host-side and
    log-based diagnosis when no MCP is configured.
- Host-side tooling referenced by the skill: Debian/Ubuntu — `iproute2`,
  `ethtool`, `lldpd`; Windows 11 — built-in PowerShell `Net*` cmdlets + `pktmon`.

## Security & scope

- The content is **vendor-neutral**; it ships **no secrets** and no specific
  network topology. Keep your API keys in environment-local config, never in
  this repo.
- The subagent is **read-only / diagnostic by default** (`Write`/`Edit`
  disallowed) and treats controller/switch/host **mutations as
  operator-approval-gated**.

## Versioning

Releases follow [SemVer](https://semver.org) via the `version` field in
`.claude-plugin/plugin.json`. Bump it to ship an update; see `CHANGELOG.md`.

## License

[MIT](LICENSE).
