# Services

This file describes where each service runs and how the services are deployed, with the reason for each decision. The configuration of each service is in its runbook in `runbooks/`.

## The NAS

- The OS is TrueNAS Scale. It is free, and ZFS is native to it.
- **Accounts.** The NAS has one human user, and service accounts that are separate from the admin account ([security.md](security.md)). Examine the permission design again if a second person stores data here.
- Media runs on the NAS, not on the Pi. The Pi 3 cannot transcode, and no configuration changes that. So Jellyfin runs on the x86 NAS, and the Pi does only light work that is always on.

## The limits of the i3-4130

- **RAM:** 16 GB (2 × 8 GB). TrueNAS needs approximately 8 GB before the containers, and ZFS uses the remaining memory as ARC. Two slots are free, and the maximum is 32 GB.
- **Transcoding:** Quick Sync does H.264 in hardware. Haswell has no hardware for HEVC, so an H.265 transcode runs on the two cores. Expect approximately one transcode stream. Direct play is not affected.

### Playback is repaired at the client, not in the library

Each reason that Jellyfin gives for a transcode is the client, which declines something: the container, the codec, the channels or HDR. A browser declines the most. A client that is based on mpv plays the same file directly. The clients are in the [Jellyfin runbook](../runbooks/jellyfin.md), and the event is in the [build log of phase 2](../build-log/phase-2.md).

This also limits what Quick Sync in the Jellyfin container can give. It does not help with HEVC decode, with tone-mapping or with burned-in subtitles.

## Containers

Containers run as TrueNAS *Custom Apps*, not with `docker compose` over SSH.

- TrueNAS owns the Docker daemon. A stack that is made outside the middleware is not in the config backup, and an upgrade can reset it.
- The Custom App form accepts plain compose YAML. So a Custom App is the same container, made from the same YAML.
- The warning of TrueNAS about third-party apps is not a reason to go around the middleware. It is a general disclaimer, and TrueNAS shows it for each custom app.

The rules for each app:

- The state goes on host paths in `NAS_Storage/apps/<service>`, never in Docker volumes.
- `apps` and `ix-apps` are opposite cases. `apps` is the dataset of the homelab: it has snapshots, and it is in the backup set. `ix-apps` is the runtime of TrueNAS (`docker`, `app_configs`, `app_mounts` and `truenas_catalog`), and it is excluded from both snapshot tiers. It is mounted hidden at `/mnt/.ix-apps`, and the Datasets page does not show it. `zfs list -r NAS_Storage` shows both.
- Each Custom App is its own compose project with its own network. Inside an app, the services reach each other by name. Between apps, they use `<NAS_IPV4>:<published-port>`. So services that must reach each other by name are one app. Caddy and Jellyfin are each their own app.
- A published port binds to `0.0.0.0`. That includes the tailnet. Do not disable the authentication of a web UI for local addresses.
- Keep the container ports at their defaults. If the container port and the published port do not agree, the connection hangs and gives no error.

### Why there is no shared Docker network

One external network for all the apps was rejected. One `docker network create` would let Caddy reach each service by name, for example `http://jellyfin:8096`. But the middleware and the config backup do not know a network that a person made by hand. If a reset of the apps removes it, each app that declares `external: true` fails to start, all at the same time.

The premise was also wrong. Each Custom App already has its own network. The real need was name resolution between the services that talk to each other, and one app for those services gives that.

Two other shapes were rejected:

- **Caddy joins the network of a different app.** Then Caddy cannot start when that app is not up.
- **The published ports bind to loopback.** The loopback of a container is the container itself, so Caddy cannot reach them.

## Pi-hole on the Pi

Pi-hole runs on the Raspberry Pi, not as a VM on TrueNAS. The DNS of the household must not depend on the machine that is busiest and that reboots most. The plan puts the Pi on the UPS ([power.md](power.md)).

- One resolver is a single point of failure, and a plain second resolver is the wrong fix. Clients divide their queries between all the resolvers that they get. So a second resolver without a filter makes the blocking fail at intervals, and it is not a fallback. The fix is a second Pi-hole, which is planned. Until then, the gap is accepted.
- The OS is Raspberry Pi OS Lite, 64-bit, installed with Raspberry Pi Imager.
- The address is fixed with a DHCP reservation on the router, not with a static configuration on the Pi.
- The wear of the SD card is the long-term risk of the Pi. Pi-hole writes each query to a database on the card, and the swap is also there. These are the possible changes, cheapest first:
  1. A shorter retention of the queries (`database.maxDBdays` in Pi-hole v6)
  2. USB boot, which is on by default on the 3B+
  3. A high-endurance card at the next flash

The configuration is in the [Pi-hole runbook](../runbooks/pi-hole.md).

## Forgejo

Forgejo on the NAS is the git remote for everything that is not public: the private notes, and the code of projects that more than one machine works on. A private repository on GitHub is hidden from the public, but not from GitHub.

The repositories of Forgejo are data, not configuration, so they are in the backup set ([backup.md](backup.md)). The configuration is in the [Forgejo runbook](../runbooks/forgejo.md).

## Alerts

TrueNAS sends its alerts through Telegram.

- E-mail was configured first, and it never delivered. Telegram is examined more often. Gmail needs an App Password (which needs 2FA) or OAuth. That is a secret that can break with no warning on a headless machine.
- The relay is external. A notifier on the NAS cannot report that the NAS is down. On the Pi, it would depend on the SD card.
- The privacy cost is accepted. The alert texts send a drive serial, the host name and the pool name to one company, over TLS.

A monitor for the health of the services is planned. Its transport for the alerts is not decided.
