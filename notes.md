# pi-privacy-stack — engineering notes

Reclaiming my own data and running privacy + network infrastructure on a Raspberry Pi 5, written out as I learn it — partly so I remember what I did and why, partly so anyone looking can see I build things and don't just talk about them. The README has the overview; this is the running journal. **Newest update is at the top.**

> **Privacy note:** public learning log. Real hostnames, internal IPs, account lists, and credentials are omitted or replaced with obvious placeholders. The *engineering* is here; the *personal data* is not.

---

## Updates (newest first)

### v1.3 — The router swap: the whole house gets on the Pi · June 19, 2026

This was the disruptive one, and the payoff one. After 1.2 I had exactly *one* device — my laptop — pointed at the Pi by hand. Tonight I made it the whole house, automatically. The one-sentence version for a non-technical friend: *before, I'd walked over to one computer and personally told it "ask the Pi for directions"; now the network itself tells every phone, TV, and laptop to do that the moment they join, and I never touch them.*

**Why I had to bridge the gateway instead of just plugging my router in behind it.** The ISP gateway locks its DNS field, so I can't tell the network "ask the Pi" from there — that's the wall that stopped me in 1.2. The fix is to demote the gateway to a dumb modem (bridge mode) and let my own router be the brain. If I'd skipped bridging and just hung my router off the gateway, I'd have had **double-NAT** — two boxes both doing address translation, stacked. That breaks port-forwarding and some VPNs and makes every future problem twice as hard to trace. Bridge mode = one brain doing NAT/DHCP/DNS, not two. The "one device can connect to a bridged gateway" limit sounds scary but it's a non-issue: the one device is my router, and the entire house hangs off *that*.

**The actual idea of the session — by hand vs. by network.** In 1.2 I typed the Pi's address into one Mac's DNS settings. That's a sticky note on one machine. Tonight the router hands that same instruction out over DHCP to *everything* that connects. The proof was deleting the manual DNS entry off my Mac and watching it *keep working* — because it now gets the Pi automatically from the router. Deleting the thing and having it still work is the whole point: I stopped configuring devices and started configuring the network. Same logic as the IP reservation — put the instruction on the router, not the device, and it survives everything downstream.

**The blank second DNS slot is the feature.** I set DNS Server 1 to the Pi and deliberately left DNS Server 2 blank. The temptation is to drop a "backup" like 8.8.8.8 there — but that's not a backup, it's a leak. Devices would quietly fall to Google any time the Pi was busy, and those queries would vanish from my log, unfiltered. No fallback = no escape hatch = everything in my query log is the *whole* truth.

**Bypass routes — what I closed and what I flagged.** Two ways a device dodges the Pi: (1) hardcoding a public DNS like 8.8.8.8 and ignoring what the router says, and (2) **DoH** — DNS hidden inside normal HTTPS (port 443) so it doesn't even look like DNS to the network. I closed the *automatic* DoH case at the router ("Prevent client auto DoH"), which stops browsers silently switching themselves over. What I could NOT do tonight: a true port-53 redirect to catch hardcoded resolvers — stock router firmware doesn't have it (that's a Merlin-firmware "DNS Director" job for later). Honest status: automatic DoH closed network-wide, Safari covered (it follows system DNS, and the Pi already blocks iCloud Private Relay's `mask` domain), and the only gap left is *manually*-enabled DoH in a browser, which I'll handle on my privacy-browser day.

**What broke / what surprised me (the troubleshooting muscle).** Two good lessons, both from doing things slightly wrong first. (1) I tried to set up my new router *before* bridging and got out of order, then unplugged the Pi mid-fumble — and the internet "died." It wasn't really dead: my Mac still had its 1.2 hand-set DNS pointed at the Pi, so killing the Pi killed the Mac's name lookups. That's a live demo of why one device hand-tied to the Pi is fragile, and why network-wide is better. Rolled back cleanly (un-bridge, breathe, restart in order) — knowing rollback was 5 minutes away is what let me stay calm. (2) The surprise win: my router auto-created a separate IoT Wi-Fi. I worried it might skip the Pi, but when I put my phone on it and checked the query log, there it was — same `192.168.50.x` range, queries logged, Private Relay getting blocked. So I got **segmentation** (sketchy IoT gadgets walled off from my trusted devices) *and* filtering. I ID'd my own phone in the log by its Apple/iCloud lookups before I even checked its IP — kind of a privacy lesson in itself about how identifiable each device's chatter is.

**The trade-off I just took on.** Going network-wide protects every device — but it also makes the Pi a **single point of failure**. If it dies, the whole house's DNS dies with it (exactly what I felt when I unplugged it). That's the cost of the convenience. It helps the "data economy" and "tracking" lines of my threat model enormously, but it adds a new operational risk I now own: the Pi has to stay up, and its backups can't live *only* on the Pi. Tonight's fresh Teleporter export is a start, but a backup sitting on the Pi's own SD card is useless if that card is what fails — a copy has to live off the box.

*Next: Phase 1 cleanup (no hardware) — credit freezes, passkeys, privacy browser + DoH-off everywhere, data-broker purge. Then Phase 2 — pulling my data fully in-house.*

### v1.2 — DNS goes live: Pi-hole + Unbound · June 17, 2026

This was the session where the Pi stopped being an idle box and started doing a job. I stood up Pi-hole and Unbound and proved the whole thing on my laptop. The clearest way to say what I'm building is the path I want every "what's the address for X?" question to travel:

```
[my device] → [Pi: Pi-hole blocks trackers → Unbound resolves it itself] → out to the root servers → answer back
```

No ISP in the middle. No Google in the middle. Right now **only my MacBook is on that path** — I pointed it at the Pi by hand to prove it works. Getting the *whole house* on it automatically is the router swap, next session. So today is one device on the rails; 1.3 is everything.

A few things that actually clicked:

**Two different "lines."** I kept picturing the Pi as sitting *between* my modem and router. It doesn't. Traffic flows device → router → modem → internet, and the Pi just hangs off the router like any other device. The Pi is on a *separate* line — the DNS line — where my device asks it for directions *before* getting on the road. And the Pi uses that same road to do its own lookups. It's a consultant on the network, not a tollbooth on the highway.

**A fixed address isn't optional.** The Pi has to live at one address forever, because every device is about to hard-code "ask the Pi at this address" into its settings. If the address drifts, all those devices are pointing at an empty lot and DNS just dies — which looks exactly like "the internet is down." Like moving a cell an Excel formula points to: everything that referenced it breaks. I pinned it with a reservation on the gateway (tied to the Pi's MAC, not its name — which is why renaming the Pi mid-session didn't break anything).

**Why Unbound, not just Pi-hole.** Pi-hole alone still *forwards* my allowed lookups to a big upstream resolver — so I'd have just traded my-ISP-sees-everything for Google-sees-everything. Unbound makes the Pi do the lookup itself, walking from the root servers down, so no single company holds my whole query history. Proved it with `dig`: first lookup ~272 ms (real legwork out to the roots), second lookup ~0 ms (served from its own cache). That slow-then-instant *is* the proof. Honest caveat I want on record: this isn't invisibility — my ISP still sees the IPs I ultimately connect to. What I removed is the one party that had my entire DNS history in one tidy place.

**Seeing the chatter was the gut-punch.** A few minutes of light browsing and my laptop alone had fired off ~94 DNS lookups — 14 of them blocked. Roughly 15% of everything my machine asked for was ads/trackers/telemetry, most of it background noise I never initiated. None of that is new; the only thing that changed is I can finally *see* it. Can't manage what you can't see.

**Did it the infrastructure-as-code way.** Ran Pi-hole in Docker so the whole setup is one version-controlled file I can rebuild from in minutes. One detail bit-then-paid-off: `network_mode: host` lets the container share the Pi's real network, which is both why Pi-hole could hand off to Unbound on `127.0.0.1` *and* why my logs show real per-device IPs instead of one anonymous Docker address.

**Two security habits I actually used.** Before piping the Docker install script into my shell, I had it read line-by-line first — it was the legit official script, but checking is the point. And I changed the gateway admin off its default password (default creds on network gear = textbook weak link). Also: don't paste secrets into a chat window. Lesson banked.

*Next: the router swap (v1.3) — bridge the gateway, bring up my own router, point the whole network's DNS at the Pi, then close the bypass routes.*

### v1.1 — Project kickoff + Pi build · June 15–16, 2026

Day one was all the un-sexy foundation stuff, and honestly the highest-impact: threat model, decisions, and locking down accounts. I moved everything into the password manager and actually looked at the damage — a scary number of reused passwords, and a chunk that had already leaked. I fixed the ones that mattered (money and identity first), turned on app-based 2FA for banking and email, and shored up my phone line against SIM-swapping. I also kicked off pulling my data back from the big platforms, which takes days, so I started it early.

Then the Pi itself. I was nervous about the hardware part and it turned out to be the smooth bit. Grounded myself (and learned my desk's metal trim isn't actually a ground — you touch a real grounded thing instead), stuck the heatsink onto the chip (dry-fit first, it's a one-shot), mounted the fan, closed it up. The fan not spinning at first freaked me out until I realized it's temperature-controlled — off when it's cool is normal.

Flashed the OS headless with SSH on, a non-default username, and a generated password. Booted it, SSH'd in. The thing that tripped me up: I kept trying to log in with my computer's username instead of the one I'd set on the Pi. The hostname is the machine's name; the username is a separate thing — once `user@host` clicked, I was in. First thing I did once in was patch the whole system, before installing anything. A brand-new box is in its most vulnerable state, and a weird username + strong password don't patch known holes — updates do (and even then, nothing's ever 100%). Start on a solid foundation so it doesn't crack later.

---

## Why I'm doing this

The honest starting point was a question: if my data is so valuable that every company wants it, why am I handing it over for free? And the flip side — so much of life runs through this stuff now (email, passwords, the little "accept all cookies" clicks) that being careless with it leaves a wide-open door.

But I didn't want this to turn into me wrapping my phone in foil and trusting no one. So I did a threat model first, before touching any hardware. The thing that clicked: most of my sensitive data isn't even in my hands — it's sitting with my bank, with big platforms, with stores I've shopped at, and I can't make their security any better. So instead of "secure the bank" (can't), it became "shrink how exposed I am inside these systems." I'm opting out of an industry, not going off the grid.

## Threat model

*The rudder for every decision below. Written before touching hardware.*

**Assets (priority order):**

1. **Financial access** — banking, savings, cards.
2. **The keys that unlock everything else** — primary email, password vault, phone number / 2FA.
3. **Identity** — preventing impersonation / fraud.
4. **Physical safety** — home address, location/movement data.
- *Accepted risk (low priority):* private photos.

**Strategic frame:** much of my sensitive data lives in third-party systems I can't secure, so the job is to minimize exposure and shrink the blast radius of *their* breaches — not pretend I control them.

**Threat actors:**

1. **The data economy** — corporations, data brokers, ad-tech monetizing me. Certain, constant, profit-driven.
2. **Opportunistic automated attackers** — credential-stuffing bots, mass phishing, malware sweeping for reused passwords. Impersonal but high-impact.
3. **Account-takeover via weak links** — SIM-swap, 2FA phishing, fraud built on cheap broker data. *(#3 is fed by #1.)*

**Vectors → mitigations:**

| Vector | Mitigation |
|--------|------------|
| Reused/weak passwords (credential stuffing) | Password manager + unique password per site |
| Email as single point of failure | App-based 2FA + passkeys, email first |
| SMS 2FA → SIM-swap | Carrier port-out protection + app-based 2FA |
| Phishing / lookalike sites | Manager won't autofill fake domains; per-signup email aliases |
| Browser & DNS tracking | Hardened browser + Pi-hole DNS blocking + Unbound recursion |
| Data brokers aggregating a fraud-ready profile | Broker removal + periodic re-checks |
| Oversharing / account sprawl | App-permission audit + account pruning |

**Out of scope (deliberately not defending against):** nation-states / law enforcement with a warrant; a determined attacker personally targeting me; physical coercion / device seizure; zero-days and breaches at the big platforms themselves (accept residual risk, shrink what they hold); total anonymity. **Goal: opt out of an industry, not go off the grid.**

## Architecture & decisions

*Why I chose each tool over the alternative — judgment, not just configuration.*

**Router / network edge — own router, ISP gateway in bridge mode.** The ISP gateway locks its DNS field, so in router mode I can't point the network at Pi-hole. Bridge mode + my own router gives full DNS control, real VLANs to isolate IoT devices from personal ones, and a multi-gig WAN port to match the plan. Wi-Fi generation wasn't the deciding factor (heavy gear is wired) — the multi-gig ports and VLANs were.

**DNS filtering — Pi-hole (live as of v1.2), run in Docker.** Network-wide ad/tracker blocking at the DNS layer, plus query-log visibility into what every device phones home to. Currently proven on one client (the laptop); whole-network coverage comes with the router swap.

**Upstream DNS — Unbound (live as of v1.2), local recursive resolver on `127.0.0.1#5335`, set as Pi-hole's only upstream.** Forwarding to a public resolver just swaps one snoop for another. Running my own recursive resolver (walking from the roots, with DNSSEC validation) keeps resolution in-house. Honest caveat: removes the single-profiler problem, not all visibility (the ISP still sees destination IPs).

**Containerization — Docker Compose (infrastructure-as-code).** The whole service is one version-controlled file, rebuildable in minutes. Used `network_mode: host` for Pi-hole because a DNS server needs the real network (port 53, real per-client IPs in logs) and so it can reach Unbound on `127.0.0.1`. Tradeoff: host networking grants broader access than strict isolation — acceptable for a service that must sit on the LAN.

**Password manager — Bitwarden, over the OS-native vault.** First-class across a mixed device fleet; open-source and audited; and self-hostable via Vaultwarden on the Pi later, to bring the vault fully in-house.

**Remote access (Phase 2) — Tailscale / WireGuard, not a vendor relay.** A tunnel I control, nothing routed through an outside company's servers.

**OS — Raspberry Pi OS Lite (64-bit), headless.** No desktop overhead on a box accessed over SSH; frees resources for services. Packet analysis later via `tcpdump`/`tshark`, opening the `.pcap` in Wireshark on a laptop.
**Did it the infrastructure-as-code way.** Ran Pi-hole in Docker so the whole setup is one version-controlled file I can rebuild from in minutes. One detail bit-then-paid-off: `network_mode: host` lets the container share the Pi's real network, which is both why Pi-hole could hand off to Unbound on `127.0.0.1` *and* why my logs show real per-device IPs instead of one anonymous Docker address.

**Two security habits I actually used.** Before piping the Docker install script into my shell, I had it read line-by-line first — it was the legit official script, but checking is the point. And I changed the gateway admin off its default password (default creds on network gear = textbook weak link). Also: don't paste secrets into a chat window. Lesson banked.

*Next: the router swap (v1.3) — bridge the gateway, bring up my own router, point the whole network's DNS at the Pi, then close the bypass routes.*

### v1.1 — Project kickoff + Pi build · June 15–16, 2026

Day one was all the un-sexy foundation stuff, and honestly the highest-impact: threat model, decisions, and locking down accounts. I moved everything into the password manager and actually looked at the damage — a scary number of reused passwords, and a chunk that had already leaked. I fixed the ones that mattered (money and identity first), turned on app-based 2FA for banking and email, and shored up my phone line against SIM-swapping. I also kicked off pulling my data back from the big platforms, which takes days, so I started it early.

Then the Pi itself. I was nervous about the hardware part and it turned out to be the smooth bit. Grounded myself (and learned my desk's metal trim isn't actually a ground — you touch a real grounded thing instead), stuck the heatsink onto the chip (dry-fit first, it's a one-shot), mounted the fan, closed it up. The fan not spinning at first freaked me out until I realized it's temperature-controlled — off when it's cool is normal.

Flashed the OS headless with SSH on, a non-default username, and a generated password. Booted it, SSH'd in. The thing that tripped me up: I kept trying to log in with my computer's username instead of the one I'd set on the Pi. The hostname is the machine's name; the username is a separate thing — once `user@host` clicked, I was in. First thing I did once in was patch the whole system, before installing anything. A brand-new box is in its most vulnerable state, and a weird username + strong password don't patch known holes — updates do (and even then, nothing's ever 100%). Start on a solid foundation so it doesn't crack later.

---

## Why I'm doing this

The honest starting point was a question: if my data is so valuable that every company wants it, why am I handing it over for free? And the flip side — so much of life runs through this stuff now (email, passwords, the little "accept all cookies" clicks) that being careless with it leaves a wide-open door.

But I didn't want this to turn into me wrapping my phone in foil and trusting no one. So I did a threat model first, before touching any hardware. The thing that clicked: most of my sensitive data isn't even in my hands — it's sitting with my bank, with big platforms, with stores I've shopped at, and I can't make their security any better. So instead of "secure the bank" (can't), it became "shrink how exposed I am inside these systems." I'm opting out of an industry, not going off the grid.

## Threat model

*The rudder for every decision below. Written before touching hardware.*

**Assets (priority order):**

1. **Financial access** — banking, savings, cards.
2. **The keys that unlock everything else** — primary email, password vault, phone number / 2FA.
3. **Identity** — preventing impersonation / fraud.
4. **Physical safety** — home address, location/movement data.
- *Accepted risk (low priority):* private photos.

**Strategic frame:** much of my sensitive data lives in third-party systems I can't secure, so the job is to minimize exposure and shrink the blast radius of *their* breaches — not pretend I control them.

**Threat actors:**

1. **The data economy** — corporations, data brokers, ad-tech monetizing me. Certain, constant, profit-driven.
2. **Opportunistic automated attackers** — credential-stuffing bots, mass phishing, malware sweeping for reused passwords. Impersonal but high-impact.
3. **Account-takeover via weak links** — SIM-swap, 2FA phishing, fraud built on cheap broker data. *(#3 is fed by #1.)*

**Vectors → mitigations:**

| Vector | Mitigation |
|--------|------------|
| Reused/weak passwords (credential stuffing) | Password manager + unique password per site |
| Email as single point of failure | App-based 2FA + passkeys, email first |
| SMS 2FA → SIM-swap | Carrier port-out protection + app-based 2FA |
| Phishing / lookalike sites | Manager won't autofill fake domains; per-signup email aliases |
| Browser & DNS tracking | Hardened browser + Pi-hole DNS blocking + Unbound recursion |
| Data brokers aggregating a fraud-ready profile | Broker removal + periodic re-checks |
| Oversharing / account sprawl | App-permission audit + account pruning |

**Out of scope (deliberately not defending against):** nation-states / law enforcement with a warrant; a determined attacker personally targeting me; physical coercion / device seizure; zero-days and breaches at the big platforms themselves (accept residual risk, shrink what they hold); total anonymity. **Goal: opt out of an industry, not go off the grid.**

## Architecture & decisions

*Why I chose each tool over the alternative — judgment, not just configuration.*

**Router / network edge — own router, ISP gateway in bridge mode.** The ISP gateway locks its DNS field, so in router mode I can't point the network at Pi-hole. Bridge mode + my own router gives full DNS control, real VLANs to isolate IoT devices from personal ones, and a multi-gig WAN port to match the
