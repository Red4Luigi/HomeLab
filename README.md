# Homelab

This repository is the documentation of my homelab: a server rack at home, built around a self-built NAS and a 1U server that was a gift. The homelab has redundant storage, a media server, DNS filtering for the full network, and power protection with a battery.

The documentation records each decision with its reason, and what broke.

**Status:** the build is in phase 2, media and self-hosted services. The NAS is built and stable. Internal HTTPS, the media server, remote access and self-hosted git run.

## The hardware

| Machine                           | Role                                                    | Always on                          |
| --------------------------------- | ------------------------------------------------------- | ---------------------------------- |
| Self-built NAS (LGA1150)          | TrueNAS Scale, 3 × 4 TB in RAIDZ1, Jellyfin, containers | Yes                                |
| Intel R1304GL4DS9 (1U, two Xeons) | Linux From Scratch, then game-server hosting            | When it is necessary, through IPMI |
| Raspberry Pi 3 Model B+           | Pi-hole, the DNS filter                                 | Yes                                |
| TP-Link TL-SG108E                 | The switch of the homelab                               | Yes                                |
| CyberPower CP900EPFCLCD           | UPS with a pure sine wave, for a clean shutdown         | -                                  |
| StarTech 15U                      | Open-frame rack with 4 posts                            | -                                  |

The full inventory and the specifications are in [docs/hardware.md](docs/hardware.md).

**Cost:** approximately €1,490 committed, and approximately €1,645 projected in total. The parts were bought during the memory and storage shortage of 2026. For that reason, DDR3 cost approximately three times its price of 2025.

## Some decisions to read

- **Three 4 TB drives in RAIDZ1, not two drives in a mirror.** One more drive doubles the usable capacity, and the workload is large sequential media. See [docs/storage.md](docs/storage.md).
- **The first board and the first PSU were both replaced.** The main power connector of the OEM board was a proprietary 14-pin connector, not ATX, and the PSU had no SATA power. The project started as a new use for an old desktop and became a new build: only the CPU cooler and two SATA cables stayed. See [docs/hardware.md](docs/hardware.md).
- **RAID is not a backup.** The pool survives a dead drive, not a deleted file. See [docs/backup.md](docs/backup.md).
- **No inbound port is open.** Internal services get real HTTPS certificates, and remote access and the public sites start their connections from the inside. See [docs/security.md](docs/security.md) and [docs/network.md](docs/network.md).
- **Media runs on the NAS, not on the Pi.** A Pi 3 cannot transcode, and no change to the network repairs that. See [docs/services.md](docs/services.md).

## How the documentation is organised

Each fact has one home. The files are divided by the type of information.

| Location | Contents | Example |
| --- | --- | --- |
| [`docs/`](docs/) | What is built now, and why. One file for each topic | "The pool is RAIDZ1 because ..." |
| [`runbooks/`](runbooks/) | How each service is configured, sufficient to build it again | The BIOS settings of the NAS |
| [`build-log/`](build-log/) | What happened, and when. One file for each phase | "A drive did not appear on port 1. I blamed two cables. It was the port." |
| [commands.md](commands.md) | Commands, in groups by the question that they answer | "Which physical disk is this?" |
| [glossary.md](glossary.md) | The terms of this documentation | "split-horizon DNS" |

The topic files in `docs/`:

| File | Topic |
| --- | --- |
| [hardware.md](docs/hardware.md) | The machines, the parts of the NAS, the case and the rack |
| [storage.md](docs/storage.md) | The pool, the drives and the datasets |
| [backup.md](docs/backup.md) | The copies of the data, snapshots and replication |
| [services.md](docs/services.md) | Where each service runs, and how containers are deployed |
| [network.md](docs/network.md) | Internal HTTPS, DNS, remote access and the public sites |
| [security.md](docs/security.md) | The protection of the NAS and of the BMC, and the accepted risks |
| [power.md](docs/power.md) | The PSU and the UPS |
| [build-checklist.md](docs/build-checklist.md) | What to examine before a step of a build |

## Related projects

Each standalone project has its own repository. This repository only links to them.

- Quiz and homepage: a public web project. The repository will follow.
- Game-server hosting: a school project on the R1304. The repository will follow.
- Linux From Scratch: build notes. The repository will follow.

The network of the house (the VLAN design, the main switch and the access points) is a separate plan. It is not part of this repository.
