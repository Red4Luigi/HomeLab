# Storage

This file describes the storage of the NAS: the pool, the drives and the datasets, with the reason for each decision. The parameters that made them are in the [TrueNAS storage runbook](../runbooks/truenas-storage.md). The backup of the data is in [backup.md](backup.md).

## The pool

The pool `NAS_Storage` is one RAIDZ1 vdev of three 4 TB drives. It survives the failure of one drive. Its `ashift` is 12, and this value was verified after the pool was made.

The usable capacity is 7.28 TiB before the overhead of ZFS (slop space and metadata). The dashboard shows a little less. The plan uses approximately 5.7 TiB. Copy-on-write needs free space, and fragmentation becomes a problem above approximately 80%.

### Why RAIDZ1, and not a mirror

The first plan was two 4 TB drives in a mirror, with 4 TB usable. RAIDZ1 with three drives won for these reasons:

- One more drive (approximately €190) doubles the usable capacity. With mirrors, 8 TB needs two 8 TB drives later (approximately €400 or more).
- ZFS cannot convert a mirror to RAIDZ1, or RAIDZ1 to a mirror. The only way is to destroy the pool and restore the data. So the layout had to be correct before the data arrived.

A resilver of RAIDZ1 reads each remaining drive in full. That is why the external backup is a requirement ([backup.md](backup.md)).

The third drive came from a different seller. Drives from one batch can have the same failure tendency.

The pool has no auxiliary vdevs: no log, cache, spare, special or dedup vdev. Dedup is off, and `lz4` compression is on.

## Drives, boot and ports

- **Drives:** Seagate IronWolf ST4000VN006, CMR. Never SMR. The ST4000VNZ06 was rejected: it has a 64 MB cache at €225, and the ST4000VN006 has a 256 MB cache at approximately €185.
- **Boot:** a separate 2.5" SSD. All three HDDs go to the pool.
- **SATA:** 4 of the 6 ports are in use. The pool is on `SATA6G_1` to `SATA6G_3`, and the boot SSD is on `SATA6G_4`. Port 1 is degraded but cleared for use ([build-checklist.md](build-checklist.md)). The two 3 Gb/s ports are not tested yet ([hardware.md](hardware.md)).
- **Port order:** ZFS does not use it. ZFS finds the members of a pool by the GUID on each disk. So the cables, the ports and also the board can change.

## Datasets

The layout was decided before the pool held any data. The boundaries of the datasets follow policy (snapshots, replication and preset), never topic.

| Dataset | Preset | Snapshots | In the backup set | Holds |
| --- | --- | --- | --- | --- |
| `media` | Generic | Daily only | No. It can be replaced | The media library for Jellyfin |
| `personal` | SMB | Daily and weekly | Yes | Personal files and the personal media collection |
| `apps` | Generic | Daily and weekly | Yes | The state of the containers (host paths). This is the configuration that makes everything else again |
| `home` | Generic | Daily and weekly | Yes | The home directories of the logins. They are not in the config export |

`ix-apps` is the runtime dataset of TrueNAS itself. It is excluded from both snapshot tiers ([services.md](services.md)).

## How to choose the preset of a dataset

The preset sets two properties. `acltype` can change later. `casesensitivity` can never change. So one question decides the preset: will a client that ignores the case of letters (Windows or macOS) ever write to this dataset? A Generic dataset can still have an SMB share.

- `personal` is SMB. Windows wrote to it during the migration. The Windows laptop or the MacBook can write to it again.
- `media`, `apps` and `home` are Generic. Only Linux writes to them. POSIX ACLs are also correct for containers.
