# Runbook: Homepage

Homepage is the dashboard of the LAN services on the NAS, behind [Caddy](caddy.md) at `homepage.donny.lu`. It is one page of `<service>.donny.lu` links with live widgets. This file and the `apps` dataset are sufficient to build it again.

The placeholders `<NAS_IPV4>` and `<PI_IPV4>` are the fixed IPv4 addresses of the NAS and of the Pi. Their real values are not in this repository.

## Structure

- The TrueNAS catalogue app *Homepage* (community train), app `1.3.22` with Homepage `v2.5.0`. It is one container that runs as `1000:1000`.
- Plain HTTP on port `30054` (the default). TLS ends at Caddy.

## Storage

| Entry | Type | Path | Access | Mounted at |
| --- | --- | --- | --- | --- |
| Config | Host Path | `NAS_Storage/apps/homepage` | ACL: USER `1000`, Full Control, with *Force Flag* on | `/app/config` |

- The folder is the full state: `services.yaml`, `widgets.yaml`, `settings.yaml` and `bookmarks.yaml`. It also has the `custom.*` files and the `docker`, `kubernetes` and `proxmox` stubs that Homepage writes at the first start.
- `services.yaml` holds the API key of each widget in plain text. Never put it in a file that git tracks. It is in each snapshot of `apps`.

## Configuration

| Setting | Value |
| --- | --- |
| Allowed Hosts | `homepage.donny.lu` |
| Mount Docker Socket | Off |
| Resources | The defaults: 2 CPUs and 4 GiB |

- **Allowed Hosts** is the list of names that Homepage answers to. It examines the `Host` header, as a protection against DNS rebinding. It is not a list of permitted clients. `http://<NAS_IPV4>:30054` gets `400 Host validation failed`. A new name, for example a tailnet name, must be in this list before it works.
- **Keep *Mount Docker Socket* off.** A read-only mount protects the socket file, not the API behind it. Each process that can talk to the socket can start a container with `/` mounted. That is root on the NAS, from a page with no login. The socket would give the status of the containers, statistics for each container and discovery by labels. The page uses none of them.

## Widgets

The page has widgets for TrueNAS, Pi-hole and Jellyfin. They call their services by the `donny.lu` name. So they depend on Caddy and on the local records of Pi-hole, and also on the service. The widget of Pi-hole is the exception: it calls `http://<PI_IPV4>`.

- **The URL of the TrueNAS widget is `https://truenas.donny.lu`, never `http://<NAS_IPV4>`.** TrueNAS revokes the key of a user the first time that it arrives over plain HTTP, and it judges the connection that it receives. So the name is safe only because the upstream of Caddy for TrueNAS is also HTTPS ([Caddy runbook](caddy.md), "Caddyfile"). The key is the key of the dedicated read-only user, not of the admin account ([TrueNAS runbook](truenas.md), "API keys").
- **Pi-hole v6 has no read-only credential.** The key of the widget is an app password (*Settings → Web interface / API*), which can do everything that the admin password can do. It goes to `http://<PI_IPV4>`, so it crosses the LAN in clear text. To serve the admin page of Pi-hole over HTTPS is planned.
- **`url` and `href` need the scheme.** Without `https://`, the widget fails with *API Error: Unexpected error*, and the link resolves as a path on `homepage.donny.lu`.
