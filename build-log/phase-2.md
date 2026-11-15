# Build log, phase 2: media and self-hosted services

This file records what happened in phase 2, with dates. The configuration is in the runbooks: [Caddy](../runbooks/caddy.md), [Jellyfin](../runbooks/jellyfin.md), [Tailscale](../runbooks/tailscale.md) and [TrueNAS](../runbooks/truenas.md). The decisions are in [`docs/`](../docs/).

## 2026-08-27 to 28: the first containers (the apps pool, Caddy and Jellyfin)

This work took two evenings.

### The dataset that almost lost its backups

**Symptom.** After I set the apps pool, the only dataset that looked new was `NAS_Storage/apps`.

**Assumption.** This build of TrueNAS gives its runtime dataset the name `apps`. So I excluded it from both snapshot tiers.

**Cause.** `apps` is my own dataset for the configuration, made on 12 August. The runtime is `NAS_Storage/ix-apps`. It is mounted at the hidden path `/mnt/.ix-apps`, so the Datasets page of the web UI does not show it. `zfs list -r NAS_Storage` showed it.

**Fix.** I corrected the exclusions before any damage. [services.md](../docs/services.md) now gives the difference.

**Next time.** My task list said "verify the name, do not assume". I verified it in the web UI, which is the one place that hides it.

### Caddy in stages, with a gap in the stages

I deployed Caddy in two stages, so that each failure had one possible cause. Stage 1 served a fixed string over HTTP. Stage 2 added the certificate. I found three things during this work:

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

- Ask the Pi directly: `dig <name> @<pi>`. This was suggested three times before I ran it.
- Read the full output. The answer was one line above the last line.
- Do not trust a "fix" that happened when nothing changed.
- DNS does not "propagate" to clients. A client asks, or it answers from its cache. What ended in the night was a TTL.

### Jellyfin did not start

**Symptom.** A restart loop: `Access to the path '/config/log' is denied`.

**Assumption.** The *Group ID* setting replaced the own group of the app.

**Cause.** `ls -ln` showed that the config directory, which I made by hand, was `root:root 755`. A process that runs as `568:3002` is in the class *other* and cannot write.

**Fix.** *Enable ACL* on that path, with USER 568 and Modify.

**Near miss.** My first attempt was GROUP 3002 with Modify on the full `apps` dataset. That is the wrong tool for one writer. It would have opened the configuration of each service to that group.

**Next time.** Ask what the directory permits for that UID, not which groups the process is in.

`apps/caddy` has the same owner and mode, `root:root 755`, and Caddy works. The reason is that Caddy runs as root. To run it as a non-root user is planned.

### A MAC address in an IPv6 address

The new `AAAA` records pointed at the SLAAC address of the NAS. The EUI-64 form of that address contains the MAC address of the interface. So a hardware identifier went into DNS. I repaired it with a static ULA. My check for private values did not find this form. It finds it now, and a planted value proved it.

### Tailscale: four faults at the same time

I built this on the evening before a trip.

1. **A typo of one digit in the nameserver of the tailnet, two times.** A wrong resolver fails with no message.
2. **I took "I can reach the NAS by its IP address" as proof that forwarding worked.** The NAS is the router. Only a different host on the LAN tests forwarding. Two rounds of diagnosis depended on this error.
3. **dnsmasq dropped the queries that did arrive:** `ignoring query from non-local network`, from the IPv6 range of Tailscale. The subnet router does NAT for IPv4, but not for IPv6.
4. **Overrides on the client.** I had to exclude Private DNS on Android and DoH in the browser.

**Next time.** Read the log of the resolver first. It was requested two times, and I read it last. One line in it gave the source address and the reason.

### The static ULA

**Symptom.** The web UI had no field for a static IPv6 address. To disable *Autoconfigure IPv6* changed nothing.

**Cause.** DHCP hides the fields for static addresses. When DHCP is off, the fields appear. But that also removes the gateway and the nameservers that DHCP supplied. Enter them again. If not, the NAS loses its route out and still looks healthy.

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

**Assumption.** The server was the problem, and Quick Sync in the Jellyfin container would repair it. It would not. The HD 4400 cannot decode HEVC, tone-mapping is a separate step, and burned-in subtitles stay on the CPU.

**Fix.** Jellyfin Media Player on the desktop. mpv handles all four natively, and the dashboard showed Direct Play. The server did not change.

**Next time.** Read the reasons and put them in order before you make a theory. The container and the channels are cheap. Bitmap subtitles and HDR each force a full new encode.
