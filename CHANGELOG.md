# Changelog

All notable changes to this plugin are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and this project
adheres to [Semantic Versioning](https://semver.org).

## [0.1.0] - 2026-05-28

### Added
- `network-engineer` subagent: read-only-by-default diagnostic persona for
  UniFi + Debian/Ubuntu + Windows 11 networking, preloading the reference skill.
- `network-reference` skill (open Agent Skills format) with progressive
  disclosure:
  - `references/unifi.md` — fabric model, STP/loops, SFP/SFP+ optics,
    Integration API, syslog.
  - `references/linux.md` — Debian/Ubuntu tooling, ARP flux, bonding, VLAN,
    macvlan, Docker Swarm.
  - `references/windows.md` — Windows 11 PowerShell net tooling, NIC advanced
    properties/EEE, LBFO teaming, VLAN, pktmon.
  - `references/diagnostics-playbook.md` — flap/loop monitor + syslog +
    correlate + bisect procedure (Bash and PowerShell).
- CI: manifest-driven SemVer release, `claude plugin validate --strict` +
  `skills-ref validate`, and CodeQL scanning of GitHub Actions workflows.
