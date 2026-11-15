# Runbook: Pi-hole

Pi-hole filters DNS for the full network, on the Raspberry Pi. This file is sufficient to build it again if the SD card dies. The reasons why it runs on the Pi and on this OS are in [services.md](../docs/services.md). The events are in the [build log of phase 3](../build-log/phase-3.md).

| Placeholder | Meaning |
| --- | --- |
| `<PI_IPV4>` | The fixed IPv4 address, reserved on the router outside the DHCP pool |
| `<PI_ULA>` | The static IPv6 ULA. Its host part mirrors the IPv4 address |
| `<ULA_PREFIX>` | The ULA /64 of the router |
| `<LAN_CIDR>` and `<ROUTER_IP>` | The subnet of the LAN, and the router, which is the DHCP server |
| `<LOCAL_DOMAIN>` | The local domain of the router |

The real values are not in this repository. The MAC addresses below show only a part (`b8:27:eb:xx:xx:xx`).

## Host

| Item | Value |
| --- | --- |
| Hardware | Raspberry Pi 3 Model B+ Rev 1.3 |
| OS | Raspberry Pi OS Lite 64-bit: Debian 13 (trixie), kernel `6.18.50+rpt-rpi-v8` |
| Memory | 905 MiB of RAM, and 904 MiB of swap (`dphys-swapfile`, on the SD card) |
| Storage | A 29 GB card, approximately 10% used |
| Pi-hole | Core v6.4.3, Web v6.6, FTL v6.7.1 |
| Network | NetworkManager, with the connection `Wired connection 1` on `eth0` |

## Before the flash

These steps apply again to a second Pi-hole.

1. In Raspberry Pi Imager, set the OS customisation: the host name, SSH with your public key, and password authentication off. Then the Pi is reachable without a monitor at the first boot.
2. Reserve the address of the Pi on the router, before a client uses it for DNS.
3. Select the protection against SD-card wear before the first boot ([services.md](../docs/services.md)).

SSH sessions from a script stop at each prompt. Use key authentication, connect one time by hand first, and use `raspi-config nonint` and `apt-get -y`.

## Network

**IPv4:** a DHCP reservation on the router. Then there is one place to look, and the address survives a new flash.

**IPv6:** a static ULA, added next to SLAAC:

```bash
sudo nmcli con mod "Wired connection 1" +ipv6.addresses <PI_ULA>/64
sudo nmcli dev reapply eth0
```

- The `+` adds to the list. Without it, `nmcli` replaces the address list.
- Keep `ipv6.method` on `auto`. Most guides say `manual`, but that stops SLAAC, the global address and the default route.
- `dev reapply` applies the change and keeps the SSH session.

To make sure, run `ip -6 addr show dev eth0`: the static address shows `valid_lft forever`. Advertise the ULA, never the global `2001:…` address. The global address changes if the ISP gives a new prefix.

## Installation

| Setting | Value | Reason |
| --- | --- | --- |
| Upstream DNS | Cloudflare `1.1.1.1` and `1.0.0.1`, and their IPv6 addresses | No filter upstream, so a blocking test has a clear result. Temporary, until Unbound replaces it (planned) |
| DNSSEC | Off | Unbound will validate |
| Blocklist | Only the StevenBlack unified list: 93,156 domains | It is curated not to break things. Add lists one at a time, each for a reason |
| Query logging | On | For the diagnosis of problems, not for the dashboard |
| Privacy level | 0 (show everything) | Examine it again, because the household uses Pi-hole now. With conditional forwarding, the log names the devices |
| Blocking mode | NULL: a blocked name answers `0.0.0.0` and `::` | |

Set the admin password with `sudo pihole setpassword`. The command asks for it. A password as an argument goes into the shell history.

## Conditional forwarding

Conditional forwarding sends local names and reverse lookups of the LAN to the router. Set it in *Settings → DNS → Conditional Forwarding*:

| Field | Value |
| --- | --- |
| Local network (CIDR) | `<LAN_CIDR>` |
| DHCP server | `<ROUTER_IP>` |
| Local domain | `<LOCAL_DOMAIN>` |

- **Add the row with the `+` button of the table before you save.** If not, *Save & Apply* does nothing, and `FTL.log` says `No config changes detected`. A real save records `Wrote config file` and `Restarting FTL`.
- The local domain of the router is under a real public TLD. Without conditional forwarding, the name resolves to the server of a stranger ([build log of phase 3](../build-log/phase-3.md)).
- If local names still fail, add an exception to the DNS rebind protection of the router. That was not necessary here.
- The host names of the devices never leave the network. The suffix is added locally when the router answers the reverse lookup.

`forwarded` in the log means "sent to an upstream". The destination is the important part. To confirm that no local name goes upstream:

```bash
# escape the dots in <LOCAL_DOMAIN>; reverse the LAN prefix: 192.168.N.0/24 → N.168.192.in-addr.arpa
sudo grep -E 'forwarded .*(<LOCAL_DOMAIN>|<REVERSED_LAN_PREFIX>\.in-addr\.arpa) to' \
  /var/log/pihole/pihole.log | grep -vF 'to <ROUTER_IP>'
```

No output means that it is clean. Reverse lookups of public addresses go upstream, and that is correct.

## Router handover

Change the LAN DNS setting of the router, which is the server that it gives to the clients. Do not change its own upstream DNS.

| Stack | Field | Value |
| --- | --- | --- |
| IPv4 | Local DNS server, which DHCP gives out | `<PI_IPV4>` |
| IPv6 | Local DNSv6 server, which the router advertisement announces | `<PI_ULA>` |

Set both stacks, or neither. With only IPv4, the clients resolve over IPv6 through the router and skip the filter. Windows prefers the IPv6 resolver.

The IPv6 DNS server arrives by router advertisement, so `ipconfig /renew` does not get it. On Windows, disable and enable the adapter:

```
netsh interface set interface "Ethernet" admin=disable
netsh interface set interface "Ethernet" admin=enable
ipconfig /flushdns
```

The other devices change when their leases renew. The ULA can appear two times in the DNS list of a client. That has no effect.

## Verification

```bash
dig @<PI_IPV4> coccyxwickimp.com +short     # blocked → 0.0.0.0
dig @<PI_ULA>  wikipedia.org     +short     # real address
pihole status
ss -tuln | grep ':53 '                      # 0.0.0.0:53 and [::]:53
```

The important test is from a client, with no server argument: `nslookup new.915yzt.cn` must give `0.0.0.0` and `::`. The real verdict is the log, not `pihole status`:

```bash
sudo grep -E 'gravity blocked' /var/log/pihole/pihole.log | tail     # compare with "forwarded … to 1.0.0.1"
```

The databases (FTL includes sqlite3):

```bash
sudo pihole-FTL sqlite3 /etc/pihole/gravity.db 'SELECT COUNT(*) FROM vw_gravity;'
sudo pihole-FTL sqlite3 /etc/pihole/pihole-FTL.db \
  "SELECT datetime(timestamp,'unixepoch'), client, domain, status FROM queries ORDER BY timestamp DESC LIMIT 10;"
```

## How to identify IPv6 clients

A device has some IPv6 addresses at the same time: stable and temporary, ULA and global, and link-local. The temporary addresses change, and phones use a random MAC address. So identify a device by its MAC address, not by its IPv6 address:

```bash
ip -6 neigh show | grep -i '<part of the address>'      # → lladdr 5e:xx:xx:xx:xx:xx
sudo pihole-FTL sqlite3 /etc/pihole/macvendor.db "SELECT mac, vendor FROM macvendor WHERE mac = 'B8:27:EB';"
```

- No vendor means a random MAC address. In a random address, bit 1 of the first byte is set (`5e`, `c2`). In a real address, it is clear (`24`, `b8`). Find such a device in the device list of the router, by its MAC address.
- An EUI-64 address shows its MAC address: `…ba27:ebff:fexx:xxxx` comes from `b8:27:eb:…`. Insert `ff:fe`, and invert that bit.
- The router publishes no reverse records for the ULA prefix. So IPv6 clients stay as addresses, and IPv4 queries get names through conditional forwarding. Expect one device with more than one identity.
- To give a Windows desktop one fixed address, run `netsh interface ipv6 set privacy state=disabled store=persistent`, and keep `randomizeidentifiers` enabled. Then give it a name in *Local DNS Records*. Do not do this on a laptop that moves between networks.

## Problems to know

- `pihole status` reports the setting, not if a blocklist is loaded. Read `pihole.log`.
- A hosts list blocks exact host names. The StevenBlack list has `mediavisor.doubleclick.net`, not `doubleclick.net`. Use test domains from the loaded list.
- DNS-over-HTTPS in a browser goes around Pi-hole, with no symptom.
- `pihole -q` needs the web password in v6, and it blocks an SSH session that is not interactive.
- **A local DNS record needs an `A` record and an `AAAA` record.** If not, the name fails at intervals ([network.md](../docs/network.md)). To examine it, run `dig A` and `dig AAAA` against the Pi, and compare the `aa` flag.
- A repaired resolver does not repair a client that has a bad answer in its cache. Run `ipconfig /flushdns` (`nslookup` does not use that cache). Clear HSTS if the wrong host served HTTPS.
- Pi-hole and a blocker in the browser work together. Pi-hole covers each device. uBlock handles ads in the page and first-party ads.
