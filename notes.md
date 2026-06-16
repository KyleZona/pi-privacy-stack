# Notes — why I'm doing this, and how it's going

**Version 1.1** — Phase 1 (network foundation): project kickoff + Pi build.

So here's the deal. I'm taking back my own data and trying to actually understand how my digital life works, and I'm doing it on a Raspberry Pi 5. This is me writing it out as I go — partly so I remember what I did and why, partly so anyone looking can see I build things and don't just talk about them.

## Why I'm even doing this

The honest starting point was a question: if my data is so valuable that every company wants it, why am I handing it over for free? And the flip side of that — so much of life runs through this stuff now (email, passwords, the little "accept all cookies" clicks) that being careless with it leaves a wide-open door.

But I didn't want this to turn into me wrapping my phone in foil and trusting no one. So I did a threat model first, before touching any hardware. Basically four questions: what am I protecting, who actually wants it, how would they get it, and what am I deliberately *not* going to worry about.

What I'm protecting, in order: my money first — bank and savings — because that's the one that would genuinely keep me up at night and stop me from taking care of my dogs. Then the things that guard the money: my main email, my password manager, my phone number (so much 2FA runs through it). Then my identity itself, so nobody can pretend to be me. Then physical stuff like my home address. And honestly, if my private photos leaked I'd shrug — so I'm not spending energy there.

The thing that clicked: most of my sensitive data isn't even in my hands. It's sitting with my bank, with big platforms, with stores I've shopped at — and I can't make their security any better. So I stopped trying to do the impossible. Instead of "secure the bank" (can't), it became "shrink how exposed I am inside these systems" — a different password everywhere so one leak doesn't cascade, app codes instead of text codes, fewer accounts, less of me floating around out there.

And who I'm actually worried about isn't some hooded hacker who's chosen me personally. It's three things: the data economy quietly monetizing me, automated attacks that sweep up reused passwords by the million, and fraud that gets built out of cheap data-broker info. Who I'm *not* worried about: nation-states, anyone personally obsessed with me, someone physically grabbing my devices. I'm not that interesting a target, and I'm not trying to vanish. I'm opting out of an industry, not going off the grid.

## The calls I made, and why

A few decisions I want on record, because the *why* matters more than the *what*.

**Router.** My ISP gateway won't let me change its DNS — which is the whole point of running Pi-hole — so I'm putting it in bridge mode and running my own router behind it. That gets me real DNS control, plus VLANs so I can segment the network and keep less-trusted devices off my actual computers, plus fast enough ports to not bottleneck my plan. Funny enough, Wi-Fi speed wasn't why I picked it — my main gear is all wired anyway. It was the ports and the VLANs.

**DNS.** I want to run Unbound so the Pi resolves DNS itself instead of forwarding everything to some big provider who'd just see all my queries anyway. Kind of defeats the purpose otherwise.

**Password manager.** Went with Bitwarden over the built-in option. Two reasons: it works everywhere (my stuff isn't all one ecosystem), and I can eventually host it myself on this same Pi. That last part is basically the whole project in miniature — own the thing that holds your most sensitive data.

**Remote access, later.** When I want to reach this from outside my house, I'll do it with my own tunnel (Tailscale or WireGuard), not some company's relay service. Same logic as everything else here.

**OS.** Went headless, no desktop. It's a server I talk to over SSH, so a desktop would just waste resources. If I ever want to dig into network traffic I'll capture it on the Pi and open it in Wireshark on my laptop.

## Building it

Day one was all the un-sexy foundation stuff, and honestly the highest-impact: threat model, decisions, and locking down accounts. I moved everything into the password manager and actually looked at the damage — a scary number of reused passwords, and a chunk that had already leaked. I fixed the ones that mattered (money and identity first), turned on app-based 2FA for banking and email, and shored up my phone line against SIM-swapping. I also kicked off pulling my data back from the big platforms, which takes days, so I started it early.

Then the Pi itself. I'll be real — I was nervous about the hardware part and it turned out to be the smooth bit. Grounded myself (and learned my desk's metal trim isn't actually a ground — you touch a real grounded thing instead), stuck the heatsink onto the chip (dry-fit first, it's a one-shot), mounted the fan, closed it up. The fan not spinning at first freaked me out for a second until I realized it's temperature-controlled — off when it's cool is normal.

Flashed the OS headless with SSH on, a non-default username, and a generated password. Booted it, SSH'd in. The one thing that tripped me up: I kept trying to log in with my computer's username instead of the username I'd set on the Pi. The hostname is the machine's name; the username is a separate thing. Once that clicked — `user@host` — I was in. Which, by the way, is way more anticlimactic than the movies make it look. No green text raining down. Just a prompt.

First thing I did once I was in was patch the whole system, before installing anything. A brand-new box is in its most vulnerable state, and a weird username and a strong password don't patch known holes — updates do (and even then, nothing's ever 100%). Start on a solid foundation so it doesn't crack on you later.

## What's next

Pi-hole, then Unbound, then swapping in the router and pointing the whole network's DNS at the Pi so it covers everything. After that, the fun Phase-2 stuff — hosting my own password vault, my own files, and other things I currently rent from big tech. One thing at a time.

---

## Version history

- **v1.1** — Project kickoff + Pi build. Threat model, tooling decisions (router, DNS, password manager, OS), account-hardening foundation (password manager migration, credential audit, app-based 2FA, SIM-swap protection), and the Raspberry Pi 5 assembled, flashed headless, patched, and SSH-secured.
