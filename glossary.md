# Glossary

This file explains the terms of this documentation. A term that has a decision behind it links to the file that gives the reason.

## Storage and ZFS

- **ZFS:** the file system of TrueNAS. It has checksums, snapshots and self-healing.
- **vdev:** a building block of a ZFS pool. Here, it is one RAIDZ1 group of three drives.
- **RAIDZ1:** an array with single parity, like RAID5. A group of n drives gives the capacity of n − 1 drives and survives one failure.
- **auxiliary vdev:** an optional vdev for a log, a cache, a spare, metadata or dedup. The pool has none ([docs/storage.md](docs/storage.md)).
- **dataset:** a file system in a pool. Snapshots, replication, shares and most properties are set for each dataset.
- **ashift:** the block alignment of a vdev. It is fixed when the vdev is made. The value `12` means 4K sectors.
- **slop space:** the 1/32 of a pool that ZFS keeps free, so that a full pool can still be written.
- **ACL:** the permission model of a dataset: POSIX (Linux) or NFSv4 (as on Windows). The preset of the dataset sets it.
- **setgid** (on a directory): new files get the group of the directory. That is why shared trees have the mode `2775`.
- **CMR and SMR:** recording types of hard drives. The pool uses CMR drives.
- **SMART:** the self-monitoring of a drive. Attributes 5, 197 and 198 decide a return. Attribute 199 counts link errors (the cable or the port, not the media). Attribute 183 is specific to the vendor.
- **badblocks:** a destructive test of the full surface of a drive.
- **LBA:** the unit in which SMART counts reads and writes.

## Network and DNS

- **DNS-01:** a challenge that proves control of a domain with a public TXT record. With it, names that are only internal get real certificates with no open port ([docs/network.md](docs/network.md)).
- **split-horizon DNS:** a name that resolves differently inside and outside the LAN.
- **NXDOMAIN and NODATA:** NXDOMAIN means "the name does not exist", and it denies each record type of that name. NODATA means "the name exists, but not this record type".
- **`aa`:** the flag for an authoritative answer. It is set when a server answered from its own records. It is absent when the server forwarded the query.
- **ULA:** the private range of IPv6 (`fd00::/8`).
- **SLAAC:** the automatic address configuration of IPv6. Its EUI-64 form puts the MAC address of the interface in the address.
- **subnet router:** a Tailscale node that forwards traffic for a full LAN.
- **exit node:** a Tailscale node that takes the default route. The NAS must never be one ([docs/network.md](docs/network.md)).
- **MagicDNS:** the name service of Tailscale. It sends resolver settings to the clients.
- **SNAT:** a change to the source address of a packet. A Tailscale subnet router does it for IPv4 by default, but not for IPv6 ([Tailscale runbook](runbooks/tailscale.md)).
- **UPnP:** a protocol with which a device on the LAN asks the router to open an inbound port for it.
- **Cloudflare Tunnel:** an outbound connection from `cloudflared` to Cloudflare. It publishes a site with no inbound port ([docs/network.md](docs/network.md)).
- **Cloudflare Access:** a login that Cloudflare enforces before a request enters the tunnel. It signs each request that it admits with a JWT, a token that the origin can verify.

## Hardware and firmware

- **BMC and IPMI:** the out-of-band controller of a server. It gives remote power control and a console, independent of the OS ([docs/security.md](docs/security.md)).
- **AHCI:** the SATA mode that ZFS needs.
- **CSM:** the compatibility layer for legacy boot.
- **LPM:** the link power management of SATA.
- **Quick Sync:** video encode and decode on the Intel iGPU. Haswell has no support for HEVC, AV1 or VP9 ([docs/services.md](docs/services.md)).
- **HDR10:** video with a wide range. Tone-mapping converts it to SDR, and that is a full new encode.
- **PGS (PGSSUB):** bitmap subtitles from Blu-ray. Text subtitles (SRT) are different: if the client cannot draw PGS subtitles, they must be burned into the video.
- **MDS:** a side-channel fault of CPUs of the Haswell period. The risk is accepted ([docs/security.md](docs/security.md)).

## Services and tools

- **NUT:** Network UPS Tools. It changes "on battery" into a clean shutdown ([docs/power.md](docs/power.md)).
- **Caddy:** the reverse proxy for `<service>.donny.lu` ([Caddy runbook](runbooks/caddy.md)).
- **Homepage:** the dashboard of the services on the LAN ([Homepage runbook](runbooks/homepage.md)).
- **Direct Play, Direct Stream and transcode:** the delivery modes of Jellyfin, cheapest first. Direct Play sends the file with no change. Direct Stream changes the container or encodes the audio again. A transcode encodes the video again. What the client accepts decides the mode.
- **mpv:** the player in Jellyfin Media Player. It plays what a browser declines.
- **Forgejo:** self-hosted git ([docs/services.md](docs/services.md)).
- **`midclt`:** the CLI of the TrueNAS middleware. It is the reliable way to see what a build supports.
- **cgroup:** the process group of systemd. The cgroup of a login session is killed at logout, so long jobs use `systemd-run`.
- **container format:** the wrapper around the streams of a media file. The metadata of Matroska (`.mkv` and `.webm`) can be edited in place with `mkvpropedit`. The metadata of MP4 is before the media, so an edit writes the full file again.
- **LFS:** Linux From Scratch. It is a related project. It will get its own repository.

## Outside this repository

- The network of the house: its main switch, the VLANs and possible routers. It is a separate plan.
- The code of the quiz and homepage project. It will get its own repository. Where it runs is in [docs/network.md](docs/network.md).
