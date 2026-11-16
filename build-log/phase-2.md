# Build log, phase 2: media and self-hosted services

This file records what happened in phase 2, with dates. The configuration is in the runbooks: [Caddy](../runbooks/caddy.md), [Forgejo](../runbooks/forgejo.md), [Homepage](../runbooks/homepage.md), [Jellyfin](../runbooks/jellyfin.md), [Tailscale](../runbooks/tailscale.md) and [TrueNAS](../runbooks/truenas.md). The decisions are in [`docs/`](../docs/).

## 2026-08-27 to 28: the first containers (the apps pool, Caddy and Jellyfin)

This work took two evenings.

### The dataset that almost lost its backups

**Symptom.** After I set the apps pool, the only dataset that looked new was `NAS_Storage/apps`.

**Assumption.** I forgot the `apps` dataset that I made on 12 August, and took it for the runtime dataset.

**Cause.** `apps` is my own dataset for the configuration. The runtime is `NAS_Storage/ix-apps`. It is mounted at the hidden path `/mnt/.ix-apps`, so the Datasets page of the web UI does not show it. `zfs list -r NAS_Storage` showed it.

**Fix.** `zfs list` found the confusion before any change to the snapshot tasks. [services.md](../docs/services.md) now gives the difference.

### Caddy in stages, with a gap in the stages

I deployed Caddy in two stages, on advice, so that each failure had one possible cause. Stage 1 served a fixed string over HTTP. Stage 2 added the certificate. This work found three things:

- The web UI of TrueNAS holds ports `80` and `443`.
- A bind mount whose source file does not exist becomes a directory.
- The stock image cannot do DNS-01, and a Custom App cannot build an image.

The gap: stage 1 tested only port 80, and stage 2 depended on port 443.

### The DNS failure that needed three diagnoses

**Symptom.** A real certificate was issued. But the internal name did not load in two browsers and on a phone. `nslookup` answered NXDOMAIN after the record existed in Pi-hole. Later, each new name did the same: it worked after a cache flush and failed again in less than one hour.

**First diagnosis, wrong.** The configuration: a missing part of the compose file, a public record that was necessary, or a port or a CNAME in the record. The configuration was correct.

**Second diagnosis, not complete.** A client asked for the name before the record existed. The stub resolver kept NXDOMAIN in its cache until it expired in the night. That explains one failure, not a failure that comes back. After `resolvectl flush-caches`, the name still failed.

**Cause.** `nslookup` sends two queries. The `A` query came back `NOERROR` with the `aa` flag: Pi-hole answered from its record. The `AAAA` query came back NXDOMAIN without `aa`: Pi-hole forwarded it to the public zone, which does not know the name. NXDOMAIN means that the name does not exist, so it also denied the `A` record that worked. It had a negative TTL of 1800 seconds, and the local `A` record had a TTL of 0.

**Fix.** An `AAAA` record for each internal name ([network.md](../docs/network.md)).

**Next time.**

- Ask the Pi directly: `dig <name> @<pi>`.
- Read the full output. The answer was one line above the last line.
- Do not trust a "fix" that happened when nothing changed.

### Jellyfin did not start

**Symptom.** A restart loop: `Access to the path '/config/log' is denied`.

**Assumption.** The *Group ID* setting replaced the own group of the app.

**Cause.** `ls -ln` showed that the config directory, which I made by hand, was `root:root 755`. A process that runs as `568:3002` is in the class *other* and cannot write.

**Fix.** *Enable ACL* on that path, with USER 568 and Modify.

**Near miss.** My first idea was GROUP 3002 with Modify on the full `apps` dataset. I stopped before I applied it, because only Jellyfin writes to those logs. GROUP 3002 would have opened the configuration of each service to that group.

**Next time.** Ask what the directory permits for that UID, not which groups the process is in.

`apps/caddy` has the same owner and mode, `root:root 755`, and Caddy works. The reason is that Caddy runs as root. To run it as a non-root user is planned.

### A MAC address in an IPv6 address

The new `AAAA` records pointed at the SLAAC address of the NAS. The EUI-64 form of that address contains the MAC address of the interface. So a hardware identifier went into DNS. I repaired it with a static ULA. My check for private values did not find this form. Now its list has this value.

### Tailscale: four faults at the same time

I built this on the evening before a trip.

1. **A typo of one digit in the nameserver of the tailnet, two times.** A wrong resolver fails with no message.
2. **Reaching the NAS by its IP address was taken as a test of forwarding.** The NAS is the router. Only a different host on the LAN tests forwarding. Two rounds of diagnosis depended on this error.
3. **dnsmasq dropped the queries that did arrive:** `ignoring query from non-local network`, from the IPv6 range of Tailscale. The subnet router does NAT for IPv4, but not for IPv6.
4. **Overrides on the client.** I had to exclude Private DNS on Android and DoH in the browser.

**Next time.** Read the log of the resolver first. I read it last, and one line in it gave the source address and the reason.

### The static ULA

**Symptom.** The web UI had no field for a static IPv6 address. To disable *Autoconfigure IPv6* changed nothing.

**Cause.** DHCP hides the fields for static addresses. When DHCP is off, the fields appear. But that also removes the gateway and the nameservers that DHCP supplied ([TrueNAS runbook](../runbooks/truenas.md)).

**Next time.** When a control is absent, and not only greyed out, look for the other setting on the form that controls it.

### Measured

| Item | Value |
| --- | --- |
| Pool use on 28 August | 1.79 TiB of 7.14 TiB (25.1%) |
| Drive temperature during the first full scan of the library (approximately 1 TB) | A constant 43 °C. The peak of the burn-in was 42 °C, so the cooling is sufficient under real load |

## 2026-09-10: a film that did not play

This was the first time that the transcode limits of the NAS had an effect ([services.md](../docs/services.md)).

**Symptom.** One film stuttered continuously in the browser. Everything else played.

**What the playback panel of Jellyfin showed.** The source was 1080p HEVC Main 10 with HDR10, AAC 7.1 audio and two PGS subtitle tracks. The output was H.264 at 12 fps, from a source of 23.976 fps. That is half of real time, so the transcode could never be fast enough. The panel gave four reasons: the container, the subtitle codec, the audio channels and the video range.

**Assumption.** A transcode problem on the server.

Quick Sync in the Jellyfin container would not repair it. The HD 4400 cannot decode HEVC, tone-mapping is a separate step, and burned-in subtitles stay on the CPU.

**Fix.** Jellyfin Media Player on the desktop, on advice. mpv handles all four natively, and the dashboard showed Direct Play. The server did not change.

**Next time.** Read the reasons and put them in order before you make a theory. The container and the channels are cheap. Bitmap subtitles and HDR each force a full new encode.

## 2026-09-22: Forgejo

I deployed Forgejo from the catalogue of TrueNAS, with its own Postgres. There were two problems at the deployment and one with git.

### The owner of the Postgres folder

**Assumption.** Postgres runs as `apps` (568), as each other app here. So its folder gets `apps:apps`.

**Cause.** The Postgres container of the catalogue runs as `999:999`. Postgres does not start unless it owns its data directory. Write access is not sufficient.

**Fix.** `chown 999:999` on the host path. The problem was found before the first start.

**Next time.** A bundled database is a second container, so it is a second user. Find the user of each container before you set the owner.

### The restart loop

**Symptom.** `mkdir: can't create directory '/var/lib/gitea/git': Permission denied`, then `docker setup failed`.

**Assumption.** Another location on the server had no access.

**Cause.** The same folder. `/var/lib/gitea` in the container is the host path of the data. *Enable ACL* was on, with no entry under it. The checkbox alone gives nothing.

**Fix.** An ACL entry: USER `568`, Modify. The [Jellyfin runbook](../runbooks/jellyfin.md) already had this entry.

**Next time.** The entry gives the access, not the checkbox. When an error names a path in the container, find its host path first.

### A second URL for the remote

**Symptom.** `git remote set-url --add forgejo <url>` gave `error: No such remote 'forgejo'`.

**Assumption.** `set-url` makes the remote.

**Cause.** `set-url` only changes a remote that exists. The repository had one remote, `origin`.

**Fix.** A second `url` on `origin` ([Forgejo runbook](../runbooks/forgejo.md)).

After that, `git remote -v` showed GitHub two times. `.git/config` had three `url` lines: GitHub, Forgejo and GitHub. I removed the extra line in the file. `set-url --delete` removes each match, so it would have removed both GitHub lines.

**Next time.** Run `git remote -v` before you change a remote.

## 2026-10-12 to 13: Homepage

I deployed Homepage from the catalogue of TrueNAS. There were three problems.

### Two fields filled from a wrong model

I examined the configuration before the first deployment. Two fields were wrong:

- **Mount Docker Socket.** I turned it on, because I thought that a read-only mount makes the access read-only. A read-only mount protects only the socket file. The full Docker API stays open, and that is root on the NAS ([Homepage runbook](../runbooks/homepage.md)).
- **Allowed Hosts.** I entered the addresses of the NAS and the Pi, because I read the field as a list of permitted clients. It is the list of names that Homepage answers to.

**Fix.** Before the deployment: the socket off, and the `donny.lu` name in *Allowed Hosts*.
### The key of the TrueNAS widget

**Symptom.** The widget showed `TrueNAS authentication failed`.

**Cause.** TrueNAS revokes the API key of a user when the key arrives over plain HTTP. It judges the connection that it receives. The widget called `https://truenas.donny.lu`, but Caddy forwarded to the HTTP port of the web UI. So the key arrived in clear text.

**Fix.** The upstream of Caddy for TrueNAS moved to the HTTPS port ([Caddy runbook](../runbooks/caddy.md)).

**Next time.** When a credential fails behind a proxy, examine what the backend receives, not what the client sends.

### The redirect to the IP address

**Symptom.** After the change to the HTTPS upstream, `truenas.donny.lu` sent the browser to the IP address and the port.

**Assumption.** The browser cache kept an old redirect.

**Cause.** A live `302` from TrueNAS. TrueNAS makes its redirect from `/` to `/ui/` from the `Host` header. To an HTTPS upstream, Caddy sends the address of the upstream as `Host`. On advice, I sent `curl` directly to TrueNAS, with and without `-H 'Host: truenas.donny.lu'`. Only the header changed, and `Location` followed it.

**Fix.** `header_up Host {host}` in the TrueNAS block.

**Next time.** Use `curl -sI` first. It has no cache, so one request shows if a redirect is old or live.
