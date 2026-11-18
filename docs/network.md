# Network

This file describes how the services are reached: on the LAN, from outside and from the internet. The security decisions are in [security.md](security.md).

The homelab shares the network of the household. The design of that network (the switches, the access points and the cables) is not part of this project.

## The line

The upstream is approximately 500 Mb/s (approximately 62 MB/s). A measurement on a schedule is planned. The NIC of the NAS is gigabit, so the WAN limits nothing on the NAS.

At line rate, an upload of approximately 1 TB takes 4 to 5 hours. For a long upload, protect the latency of the household: set a limit below the line rate. Traffic shaping and QoS are part of the household network, not of the homelab.

## Internal services: HTTPS on `donny.lu`

Each internal service has a real HTTPS certificate on a name of the form `<service>.donny.lu`.

- Caddy is the reverse proxy for these names ([Caddy runbook](../runbooks/caddy.md)).
- Pi-hole resolves the names to the LAN address of the NAS. This is split-horizon DNS: the names resolve only on the LAN, so nothing is exposed.
- Caddy gets its certificates from Let's Encrypt with the DNS-01 challenge. That challenge needs only a temporary public TXT record. Nothing at home gets a connection from outside.

### Each internal name needs an `A` record and an `AAAA` record

Give each internal name an `AAAA` record together with its `A` record. Without the `AAAA` record, the name fails at intervals:

1. Pi-hole forwards a query for a record type that it does not hold.
2. The public zone answers the `AAAA` query with NXDOMAIN. That answer denies the full name, not only the record type.
3. The answer stays in the cache for 1800 seconds. The local `A` record has a TTL of 0.

So the name works after a cache flush, and it fails again in less than one hour. This problem cost two days ([build log of phase 2](../build-log/phase-2.md)).

The `AAAA` record points at a static ULA of the NAS. Do not use the SLAAC address, because it contains the MAC address. The configuration is in the section *Network* of the [TrueNAS runbook](../runbooks/truenas.md).

Two other fixes were rejected:

- **Pi-hole as the authority for each name, for all record types.** It would stop the problem with no IPv6 in DNS. But it depends on the internal record handling of Pi-hole, which can change between versions. The internal names would also stay IPv4-only permanently. The static ULA was one change to the interface.
- **Pi-hole as the authority for all of `donny.lu`.** Then Pi-hole would also answer for the public apex, which the tunnel needs.

### Do not disable IPv6 on the host to repair a container

One container needed IPv6 off. Its documentation gives "disable IPv6 for the host" as its first option. On this NAS, that option would do damage:

- It removes the `AAAA` records that each internal name needs.
- It breaks the Tailscale setup for IPv4 and IPv6.
- It breaks the static ULA.

`sysctls` for that container only repaired it, with no side effects.

## Remote access: Tailscale

Tailscale gives personal access from outside to the web UI, the shares and Jellyfin. The NAS is a subnet router: it advertises the LAN to the tailnet ([Tailscale runbook](../runbooks/tailscale.md)).

- Remote clients use subnet routing. The `*.ts.net` names need no routing and no approval. But they give two naming schemes, one at home and one away, for each bookmark, client and runbook. Subnet routing costs more setup one time: the approval of the route, forwarding for IPv4 and IPv6, and a DNS asymmetry that the runbook describes. After that, a new service costs nothing.
- Never make the NAS an exit node. An exit node takes the default route from each other outbound path. To advertise subnet routes is safe.
- **DNS.** Pi-hole is the global nameserver in the admin console of Tailscale. Then MagicDNS does not conflict with the resolver of the host.

## Public sites: Cloudflare Tunnel

This part is planned. No site is published yet.

- The public sites will run on the NAS, not on the Pi. The Pi is the only resolver of the household. A bad deploy, a traffic spike or an out-of-memory error on its 1 GB of RAM would stop DNS for everybody. Web writes would also add wear to the SD card.
- The public sites will face the internet only through `cloudflared`. A tunnel that only connects out removes the attack surface of the network, but not that of the application. So the sites get their own dataset, the container gets no other mount, and it runs as a non-root user.
- The tunnel goes directly to each public service, never through the Caddy of the LAN. That Caddy holds the Cloudflare API token, and it serves the TrueNAS web UI and Forgejo. One wrong site block would publish an admin interface. That Caddy also adds nothing here: `cloudflared` routes by host name, and Cloudflare holds the public certificate. A static site still needs a web server, because `cloudflared` only forwards. That server is a second, minimal Caddy on plain HTTP, without the token.
- Caddy is not exposed directly. An open port with an `AAAA` record breaks the rule of no inbound port. It publishes a home address that changes, so it needs dynamic DNS. Visitors with only IPv4 cannot reach it.
- A public site that needs a login gets Cloudflare Access. Access examines the identity at the edge of Cloudflare, so a request with no login never reaches the home network. It is free for up to 50 users. Access fails open: a host name with no Access application is public, with no error. So `cloudflared` also validates the token of Access (`originRequest.access`). Then a missing policy gives a 403.
- Do not put Jellyfin behind Cloudflare. The CDN terms of Cloudflare do not permit video that is hosted outside Cloudflare. Remote media goes through Tailscale.

## Separation from the household network

The homelab is not separated from the other devices of the household. VLANs on the shared network were dropped, because they need changes to a network that the homelab only shares. A router that only the homelab uses is a possible replacement. That decision is open.
