# Runbook: Tailscale

Tailscale gives remote access to the LAN, with the NAS as a subnet router. The reasons for Tailscale, and for the rule against an exit node, are in [network.md](../docs/network.md). The events are in the [build log of phase 2](../build-log/phase-2.md).

| Placeholder | Meaning |
| --- | --- |
| `<LAN_CIDR>` | The subnet of the LAN |
| `<ULA_PREFIX>` | The ULA /64 of the router |
| `<PI_IPV4>` | The fixed IPv4 address of the Pi |

The real values are not in this repository.

## Structure

A remote client routes into the LAN. So `<service>.donny.lu` works the same as at home, with one set of names.

Never advertise this machine as an exit node. An exit node takes the default route from the other outbound paths and from LAN access. In the form, the option is next to the subnet routes.

## Host and storage

- Tailscale comes from the app catalogue of TrueNAS. Never install it on the host, where changes do not survive updates.
- **Userspace networking must be off.** A userspace node cannot route for other machines.
- The state is on a host path, `NAS_Storage/apps/tailscale`. It holds the identity of the node. If it is lost, the tailnet gets a new machine.

## Configuration

| Setting | Value | Note |
| --- | --- | --- |
| Auth Key | From the admin console | |
| Auth Once | On | Use the stored state again |
| Reset | Off | It would erase the state at each start |
| Accept DNS | Off | The NAS keeps its own resolver ([network.md](../docs/network.md)) |
| Userspace | Off | A requirement for routing |
| Accept Routes | Off | This node does not need the routes of other nodes |
| Advertise Exit Node | Off | Never |
| Advertise Routes | `<LAN_CIDR>`, `<ULA_PREFIX>` | Prefixes, not host addresses |

Forwarding must be on for IPv4 and for IPv6. Set both in *System Settings → Advanced → Sysctl*. Do not use `sysctl -w`, because its change is not permanent. To make sure, run `sysctl net.ipv4.ip_forward net.ipv6.conf.all.forwarding`. With only IPv4 on, the IPv6 route is advertised and approved, and it forwards nothing.

## Admin console

1. Approve the routes. Nothing on the NAS shows that a route waits for approval.
2. Disable the key expiry for this node. If not, the node leaves the tailnet after six months.
3. Set the global nameserver to `<PI_IPV4>`, with *Override DNS servers* on. This setting resolves the internal names for remote clients.

## DNS

The chain is: client, tailnet DNS, then Pi-hole through the subnet route. So routing must work first. A DNS failure and a routing failure look the same.

- **Set only the IPv4 nameserver.** The subnet router does source NAT for IPv4, so the queries reach Pi-hole from a LAN address. IPv6 gets no NAT. An IPv6 query arrives from a tailnet address, and dnsmasq drops it (`ignoring query from non-local network …`, recorded one time). With both nameservers configured, a client uses one or the other and fails at intervals. The alternative is a wider listening mode of Pi-hole. But an open resolver is dangerous wherever it can be reached.
- The DNS settings go to a client when it connects. After a change, turn Tailscale off and on again on the client.

## Verification

- Test a different host on the LAN, for example the admin page of Pi-hole. The NAS is the router, so a connection to the NAS proves nothing about forwarding.
- Test from a phone on mobile data, with Wi-Fi off.
- `curl -v https://<service>.donny.lu 2>&1 | grep -i 'connected to'` shows the address family in use. Expect IPv4: RFC 6724 gives ULAs (precedence 3) a lower rank than IPv4 (35).

## Problems to know

- Paste nameserver addresses. Never type them. A wrong address gives silence. Two typos of one digit cost one evening here.
- Private DNS on Android and DoH in a browser override the resolver that Tailscale gives.
- The remote path of Jellyfin is Tailscale, not Cloudflare Tunnel ([network.md](../docs/network.md)).
