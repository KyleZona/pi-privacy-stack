# pi-privacy-stack — engineering notes

Reclaiming my own data and running privacy + network infrastructure on a Raspberry Pi 5, written out as I learn it — partly so I remember what I did and why, partly so anyone looking can see I build things and don't just talk about them. The README has the overview; this is the running journal. **Newest update is at the top.**

> **Privacy note:** public learning log. Real hostnames, internal IPs, account lists, and credentials are omitted or replaced with obvious placeholders. The *engineering* is here; the *personal data* is not.

---

## Updates (newest first)

### v1.2 — DNS goes live: Pi-hole + Unbound · June 17, 2026

This was the session where the Pi stopped being an idle box and started doing a job. I stood up Pi-hole and Unbound and proved the whole thing on my laptop. The clearest way to say what I'm building is the path I want every "what's the address for X?" question to travel:

```
[my device] → [Pi: Pi-hole blocks trackers → Unbound resolves it itself] → out to the root servers → answer back
```

No Comcast in the middle. No Google in the middle. Right now **only my MacBook is on that path** — I pointed it at the Pi by hand to prove it works. Getting the *whole house* on it automatically is the router swap, next session. So today is one device on the rails; 1.3 is everything.

A few things that actually clicked:

**Two different "lines."** I kept picturing the Pi as sitting *between* my modem and router. It doesn't. Traffic flows device → router → modem → internet, and the Pi just hangs off the router like any other device. The Pi is on a *separate* line — the DNS line — where my device asks it for directions *before* getting on the road. And the Pi uses that same road to do its own lookups. It's a consultant on the network, not a tollbooth on the highway.

**A fixed address isn't optional.** The Pi has to live at one address forever, because every device is about to hard-code "ask the Pi at this address" into its settings. If the address drifts, all those devices are pointing at an empty lot and DNS just dies — which looks exactly like "the internet is down." Like moving a cell an Excel formula points to: everything that referenced it breaks. I pinned it with a reservation on the gateway (tied to the Pi's MAC, not its name — which is why renaming the Pi mid-session didn't break anything).

**Why Unbound, not just Pi-hole.** Pi-hole alone still *forwards* my allowed lookups to a big upstream resolver — so I'd have just traded Comcast-sees-everything for Google-sees-everything. Unbound makes the Pi do the lookup itself, walking from the root servers down, so no single company holds my whole query history. Proved it with `dig`: first lookup ~272 ms (real legwork out to the roots), second lookup ~0 ms (served from its own cache). That slow-then-instant *is* the proof. Honest caveat I want on record: this isn't invisibility — my ISP still sees the IPs I ultimately connect to. What I removed is the one party that had my entire DNS history in one tidy place.

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

**Router / network edge — own router, ISP gateway in bridge mode.** The ISP gateway locks its DNS field, so in router mode I can't point the network at Pi-hole. Bridge mode + my own router gives full DNS control, real VLANs to isolate IoT devices from personal ones, and a multi-gig WAN port to match the
