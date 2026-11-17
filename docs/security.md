# Security

This file describes how the homelab is protected, and the risks that are accepted. The paths in and out of the network are in [network.md](network.md).

## No inbound port

No inbound port is open, and none is necessary. Each path starts a connection from the inside:

- Tailscale, for personal access to the web UI, the shares and Jellyfin
- Cloudflare Tunnel, for the public sites (planned)

There is no port forward, no DMZ and no dynamic DNS. So the router is never part of the threat model.

This state can change with no decision from a person:

- A device can ask the router to open a port for itself (UPnP).
- IPv6 has no NAT. Only the IPv6 firewall of the router is between the internet and the global address of each device.

Only a scan from outside proves that no port is open. The router settings and the public IPv4 address are examined. A scan of the global IPv6 address of the NAS is planned.

## The NAS

- SSH accepts only keys.
- TrueNAS gets its updates.
- The web UI is never exposed to the internet. It is reachable only from the LAN and through Tailscale.
- The service accounts are separate from the admin account.

### Passwordless sudo, with a passphrase on the SSH key

These two are one decision. The admin account has passwordless sudo, and the SSH key has a passphrase.

- The sudo password stopped only a person who has the SSH key but not the password. The same password already gives control equal to root in the web UI.
- A shell as that user can also capture the password the next time that it is used.
- The passphrase is typed one time in each session, through `ssh-agent`. It also protects the key at rest.

Passwordless sudo with a key that has no passphrase would be worse than each of the two options. The configuration is in the [TrueNAS runbook](../runbooks/truenas.md).

### No host firewall on the NAS

The NAS has no firewall that a person wrote by hand. On TrueNAS, the rules would not stay: rules that are written outside the middleware are not in the config backup, and an update can remove them.

The Pi runs plain Debian, so a host firewall is planned for it.

### MDS on the i3-4130

The kernel gives this warning: `MDS CPU bug present and SMT on, data leak possible`. The risk is accepted, and SMT stays on.

- An attack needs an attacker who already runs code on the machine. A NAS with one tenant does not give that.
- The fix, to disable SMT, halves the threads of a CPU with 2 cores and 4 threads.

On a host with many tenants, the answer is the opposite. Examine this decision again if the NAS ever runs the workloads of a different person.

## The Cloudflare account

The Cloudflare account can point the tunnel at anything that the NAS can reach: the TrueNAS web UI, the router or Pi-hole. So the account is an admin credential for the LAN, and it has 2FA.

One decision is open: where the ingress rules of the tunnel are. They can be in the dashboard of Cloudflare, or in a config file on the NAS. The answer decides what a person inside the account can change.

## The BMC of the R1304

The BMC (IPMI) is the largest attack surface of the homelab. It is a computer that is always on, with its own network stack. It has full control of the power and of the console of the R1304. BMCs are known for weak defaults and old firmware.

These steps are planned before the server does any work:

1. Change the default credentials.
2. Update the firmware.
3. Keep the BMC on a management segment that only the admin machine can reach.

Never expose the BMC to the internet. Do not make a port forward to it.

The management segment needs a router that only the homelab uses. That router is an open decision. VLANs on the shared network were dropped, because they need changes to a network that the homelab only shares. Until the segment exists, a firewall must keep untrusted devices away from the BMC.
