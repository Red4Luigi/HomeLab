# Build log, phase 3: power protection and DNS

This file records what happened in phase 3, with dates: the problems, the wrong assumptions and the measurements. The configuration is in the [Pi-hole runbook](../runbooks/pi-hole.md), and the decisions are in [`docs/`](../docs/).

## 2026-07-27 to 28: Pi-hole, from the flash to the full household

I flashed the Pi 3B+ with Raspberry Pi OS Lite 64-bit, installed Pi-hole v6.4.3 and configured conditional forwarding. Then the router gave the full household to Pi-hole, for IPv4 and IPv6. It worked from end to end in one session. There were four problems.

### The card reader died during the write

The reader did not read any card. It survived a reboot. Then it failed during the write of the OS and disappeared from Disk Management.

**Assumption.** A dead card.

**Cause.** The reader. I tried another card, then another adapter. Then I tested the card in a laptop, and it read correctly. The reader had already failed before the write. A write of some GB at USB 2.0 speed is the heaviest continuous load that a cheap reader gets. A reader that loses power during a write is also a way to destroy a card. I flashed the card from the laptop.

### FTL served an empty blocklist after a clean installation

**Symptom.** The installation completed with no error. `pihole status` reported that blocking was on, and that it listened on IPv4 and IPv6. The upstream had no filter, and I also turned off the ad blockers on my own PC, so that a block could come only from the Pi. Nothing was blocked.

**Assumption.** Three theories came up, in this order. All three were wrong, and none of them was mine:

1. **A bad test domain.** This was half true. The StevenBlack list has `mediavisor.doubleclick.net`, but not the apex `doubleclick.net`. A hosts list blocks exact host names, not a domain and its children. So the obvious test domain gives a false negative. But real domains from the loaded list were not blocked either.
2. **A race at the start.** The `ActiveEnterTimestamp` of FTL was `22:23:04`, and the mtime of `gravity.db` was `22:23:07`. This was near the truth, but the evidence did not support it. I asked for a recheck, because two timestamps were weak evidence. That timestamp is the moment when systemd marked the unit as active, not the moment when FTL loaded gravity. A normal installation also signals FTL when gravity is complete.
3. **A missing config.** `FTL.log` had exactly one line: `Config file /etc/pihole/pihole.toml not available (r)`. That is the normal message of FTL at the first boot, before it writes its own defaults. The file existed, with 69 KB and the same timestamp.

**Cause.** Each fact on the disk was correct: `[dns.blocking] active = true`, 93,156 domains in `vw_gravity`, and the adlist with `enabled=1 status=1`. But `pihole.log` recorded `forwarded … to 1.0.0.1` for each query. The list in the memory of FTL was empty.

**Fix.** `sudo systemctl restart pihole-FTL`. After that: `gravity blocked coccyxwickimp.com is 0.0.0.0`.

**Next time.** Read `pihole.log` first. Its verdict for each query is the truth, and one command would have removed theories 1 and 3. `pihole status` reports the setting, not if a list is loaded. The two are independent, and the web UI shows only one of them.

### The local name of the router resolved to the server of a stranger

**Symptom.** After I pointed a PC at Pi-hole, I could not reach the router by its name.

**Assumption.** The name no longer resolved.

**Cause.** The name resolved correctly, to a real public host. The local domain of the router is under a TLD that was delegated to the public later. The vendor used it before that. With Pi-hole as a forwarder, the query left the network and got a real answer: `NOERROR`, one answer and no symptom. A router login on that page would have sent the credentials to a third party.

**Fix.** Conditional forwarding: the local names and the reverse lookups of the LAN go to the router, not upstream. One more benefit: the reverse lookups of the LAN now resolve, so the query log shows host names and not only IP addresses.

### The repaired resolver did not repair the client

**Symptom.** Pi-hole answered the name of the router correctly. One PC still went to the public host.

**Assumption.** First HSTS, but a test in a private window excluded it. Then two other theories: a WPAD proxy configuration from the time when the domain pointed to the public host, and DoH in Firefox. All three were wrong. DoH was clearly off.

**Cause.** Two independent leftovers on the client. Both survived the fix on the server.

- **The DNS cache of Windows** still held the public `A` and `AAAA` records, with approximately 15 minutes of TTL left. `nslookup` does not use this cache. So it reported the correct answer all the time, and each real application used the old one. `ipconfig /displaydns` showed the truth immediately.
- **HSTS in the browser,** recorded from the public host. It forced HTTPS on a router that serves plain HTTP.

**Fix.** `ipconfig /flushdns` and *Forget About This Site*. Both were necessary.

**Next time.** The test in a private window seemed to clear HSTS. But a private window uses the same DNS cache of the OS, so the test failed on the DNS fault before it reached HSTS. With two faults at the same time, a test of one difference gives a false negative.

The cheapest check was "does `http://<router-ip>` load?". It loaded early, but I did not report it, because it looked too obvious. That fact would have halved the search immediately.

### Measured

| Item | Value |
| --- | --- |
| Memory at idle | 148 MiB used of 905 MiB. Approximately 756 MiB free with Pi-hole in operation |
| Gravity | 93,156 domains, and a `gravity.db` of 5.5 MB |
| Card | 29 GB, approximately 10% used |
| Build of the blocklist | Approximately 3 seconds |

These numbers answer the question about Unbound: the resources of this machine are not a limit.
