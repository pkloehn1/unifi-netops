# Changelog

All notable changes to this plugin are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and this project
adheres to [Semantic Versioning](https://semver.org).

## [0.5.0](https://github.com/pkloehn1/unifi-netops/compare/v0.4.0...v0.5.0) (2026-06-02)


### Features

* **skill:** add Swarm daemon restart-policy/monitor-race case study + agent invariant ([#15](https://github.com/pkloehn1/unifi-netops/issues/15)) ([e60f704](https://github.com/pkloehn1/unifi-netops/commit/e60f7044d6ce7b7e8921dfbaab6564c7aa8087a0)), closes [#14](https://github.com/pkloehn1/unifi-netops/issues/14)

## [0.4.0](https://github.com/pkloehn1/unifi-netops/compare/v0.3.0...v0.4.0) (2026-05-30)


### Features

* **marketplace:** add marketplace.json so the repo is installable as a single-plugin marketplace ([#12](https://github.com/pkloehn1/unifi-netops/issues/12)) ([9efa127](https://github.com/pkloehn1/unifi-netops/commit/9efa127e9b7d506e4510351e8e1295572bde8497)), closes [#11](https://github.com/pkloehn1/unifi-netops/issues/11)

## [0.3.0](https://github.com/pkloehn1/unifi-netops/compare/v0.2.0...v0.3.0) (2026-05-30)


### Features

* **skill:** add example-workflows reference with four worked case studies ([#7](https://github.com/pkloehn1/unifi-netops/issues/7)) ([4a9afd6](https://github.com/pkloehn1/unifi-netops/commit/4a9afd629fba9a33f4128699f12339faf2a37ddd)), closes [#6](https://github.com/pkloehn1/unifi-netops/issues/6)


### Bug Fixes

* **ci:** use GitHub App token for release-please so REST commits are auto-signed ([#10](https://github.com/pkloehn1/unifi-netops/issues/10)) ([e41199d](https://github.com/pkloehn1/unifi-netops/commit/e41199de0d7bfbea7e440c148deaddca48eb23a9)), closes [#9](https://github.com/pkloehn1/unifi-netops/issues/9)

## [0.2.0](https://github.com/pkloehn1/unifi-netops/compare/v0.1.0...v0.2.0) (2026-05-30)


### Features

* initial unifi-netops plugin — network-engineer agent + network-reference skill ([b253580](https://github.com/pkloehn1/unifi-netops/commit/b2535806666f15c72ac488dcd9f6af4791928819))

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
