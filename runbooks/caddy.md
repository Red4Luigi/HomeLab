# Runbook: Caddy

Caddy is the reverse proxy for each LAN service at `<service>.donny.lu`, with real Let's Encrypt certificates and no inbound port. This file and the `apps` dataset are sufficient to build it again. The reasons for this design (DNS-01, split-horizon DNS, outbound connections only) are in [network.md](../docs/network.md). The events are in the [build log of phase 2](../build-log/phase-2.md).

The placeholders `<NAS_IPV4>` and `<NAS_ULA>` are the fixed addresses of the NAS. Their real values are not in this repository.

Never put the Cloudflare API token in a file that git tracks, or in the YAML of the app.

## Host

Caddy is a TrueNAS Custom App ([services.md](../docs/services.md)). The image is `ghcr.io/caddybuilds/caddy-cloudflare`: Caddy with the module `caddy-dns/cloudflare` built in.

The official image has no DNS modules. A Custom App pulls an image and cannot build one. So an image that is already built is the only option.

## Prerequisites

- Move the web UI of TrueNAS off ports `80` and `443` first. In *System Settings → General → GUI*, set `8080` and `8443`. Make sure that SSH works before you change it.
- A Cloudflare API token for the zone `donny.lu`, with `Zone:DNS:Edit` and `Zone:Read`. The template *Edit zone DNS* has both. A token that a person made by hand, without `Zone:Read`, fails as an authentication error.

## Deployment

```
/mnt/NAS_Storage/apps/caddy/Caddyfile      <- make it first: a bind mount with no source file becomes a directory
/mnt/NAS_Storage/apps/caddy/caddy.env      <- CLOUDFLARE_API_TOKEN=...
/mnt/NAS_Storage/apps/caddy/data/          <- certificates and the ACME account. It must be permanent
/mnt/NAS_Storage/apps/caddy/config/
```

```yaml
services:
  caddy:
    image: ghcr.io/caddybuilds/caddy-cloudflare:latest
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"          # HTTP/3
    env_file:
      - /mnt/NAS_Storage/apps/caddy/caddy.env
    volumes:
      - /mnt/NAS_Storage/apps/caddy/Caddyfile:/etc/caddy/Caddyfile:ro
      - /mnt/NAS_Storage/apps/caddy/data:/data
      - /mnt/NAS_Storage/apps/caddy/config:/config
```

- The token stays in `caddy.env`. A token in the YAML goes into the database of the middleware and into each config backup.
- If `data/` is not permanent, Caddy requests its certificates again at each restart. Then it reaches the weekly rate limits of Let's Encrypt.
- Caddy runs as root: no user is set, and root is the default of the image. To run it as a non-root user is planned.

## Caddyfile

```caddyfile
{
	acme_dns cloudflare {env.CLOUDFLARE_API_TOKEN}
}

truenas.donny.lu {
	reverse_proxy https://<NAS_IPV4>:8443 {
		header_up Host {host}
		transport http {
			tls_insecure_skip_verify
		}
	}
}

jellyfin.donny.lu {
	reverse_proxy http://<NAS_IPV4>:30013
}
```

The live file has one block for each service. It is on `apps`, so it is in the backup.

- The upstreams are plain HTTP, except TrueNAS. TLS ends at Caddy.
- TrueNAS revokes an API key that arrives over plain HTTP, and it judges the connection that it receives, which is the connection of Caddy. So its block goes to the HTTPS port `8443` and skips the verification ([TrueNAS runbook](truenas.md), "API keys").
- The TrueNAS block also needs `header_up Host {host}`. To an HTTPS upstream, Caddy sends the address of the upstream as `Host`. TrueNAS makes its redirect from `/` to `/ui/` from that value, and it would send the browser to the IP address and the port.
- An upstream is `<NAS_IPV4>:<published-port>`, never `localhost` or a container name. Each app has its own network, and `localhost` in Caddy is Caddy.

## How to add a service

1. Publish the port of the service, and record it.
2. Add a site block with the upstream `http://<NAS_IPV4>:<port>`.
3. In *Local DNS records* of Pi-hole, add `<service>.donny.lu` with `<NAS_IPV4>`.
4. Add the same name with `<NAS_ULA>`.
5. Restart Caddy.

- Always add both records. With only an `A` record, the name fails at intervals ([network.md](../docs/network.md), and "Problems to know" in the [Pi-hole runbook](pi-hole.md)).
- Add the record before a client asks for the name. If not, the NXDOMAIN answer stays in a cache. `resolvectl flush-caches` clears it.
- Do not use a wildcard (`address=/donny.lu/…`). It would take the full zone on the LAN, with the public apex that the tunnel uses.

## Verification

1. Ask the Pi directly for the name.
2. Load the site, and make sure that there is no certificate warning.
3. Do the same from a second device, not the device on which you configured it.

## Notes

- DNS-01 writes a temporary TXT record, and Caddy deletes it after the certificate is issued. No permanent public record exists for an internal name. Open ports have no effect on the issue of a certificate.
- Each site has its own certificate. So each internal host name is in the public Certificate Transparency logs, but the names are not reachable from the internet.
- A service behind the proxy can need its public name as a setting, for example *Published Server URL* in Jellyfin.
- The public sites will use Cloudflare Tunnel, which is a separate path. Jellyfin stays off that path ([network.md](../docs/network.md)).
