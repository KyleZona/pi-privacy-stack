# pi-privacy-stack

Reclaiming my own data and running privacy + network infrastructure on a Raspberry Pi 5 — documented openly as I learn.

This repo is the engineering journal for an ongoing project: shrink my exposure to the data-broker / ad-tech economy, harden the accounts that matter, and self-host the services I currently rent from big tech — starting with network-wide DNS filtering and growing from there.

> **Note on privacy:** This is a public learning log. Personal identifiers, real hostnames, internal IPs, account lists, and credentials are deliberately omitted or replaced with obvious placeholders. The *engineering* is here; the *personal data* is not.

## Why

A threat-model-first approach: I'm defending the data that lets me function — financial access first, then the keys that guard it (email, password vault, phone/2FA) — against the data economy, opportunistic automated attacks, and account-takeover fraud. I'm not trying to disappear; I'm opting out of an industry. The full reasoning, decisions, and build story are in [`notes.md`](notes.md).

## Architecture (planned)

```
ISP gateway (DNS-locked)  ->  [bridge mode]  ->  own router (multi-gig WAN, VLANs)
                                                   |
                                                   +-- wired devices (switch)
                                                   +-- Raspberry Pi 5  ->  Pi-hole + Unbound (network DNS)
```

- The ISP gateway can't set custom DNS, so it runs in **bridge mode** behind a router I control.
- **Pi-hole** becomes the network's DNS (ad/tracker blocking + query visibility); **Unbound** makes it a self-contained recursive resolver so no third party sees my queries.
- A **VLAN** segments less-trusted devices off from personal ones.

## Progress

- [x] Threat model written
- [x] Architecture + tooling decisions
- [x] Account-hardening foundation (password manager, credential audit, app-based 2FA)
- [x] Raspberry Pi 5 assembled, headless OS, patched + SSH-secured
- [x] Pi-hole + Unbound live (Docker; recursive resolver with DNSSEC) — proven on one client
- [x] Router bridge + network-wide DNS — every device on the Pi's DNS path (**Phase 1 complete**)
- [ ] Phase 1 cleanup — passkeys, credit freezes, privacy browser, broker purge
- [ ] Phase 2 self-hosting (roadmap below)

## Roadmap

**Phase 1 — Network foundation:** Pi-hole, Unbound, router bridge, network-wide DNS.

**Phase 2 — Self-hosting:** Tailscale/WireGuard (own remote access), Vaultwarden (own password vault), Nextcloud (files/photos/calendar), FreshRSS/SearXNG. NVMe upgrade before DB-heavy services.

## Read more

[`notes.md`](notes.md) — the why, the decisions, and the dated build journal, in my own words.
