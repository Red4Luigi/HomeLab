# Runbook: TrueNAS Scale, the storage

This file records the pool, the datasets, the permissions and the data protection of the NAS, sufficient to build them again. The host is in the [TrueNAS runbook](truenas.md). The reasons are in [storage.md](../docs/storage.md) and [backup.md](../docs/backup.md).

The pool `NAS_Storage` (3 × 4 TB in RAIDZ1) was made on 11 August 2026.

| Placeholder | Meaning |
| --- | --- |
| `<NAS_IPV4>` | The fixed IPv4 address of the NAS |
| `<SMB_USER>` | The account without privileges that is only for SMB |

The real values are not in this repository. The map from bay to serial to port is also not in this repository.

## Ports

| Port | Device | Note |
| --- | --- | --- |
| `SATA6G_1` | IronWolf | A degraded port that the burn-in cleared. Do not change its cable (see below) |
| `SATA6G_2` | IronWolf | |
| `SATA6G_3` | IronWolf | |
| `SATA6G_4` | Intenso SSD | Boot. It is the one device that SATA II would slow down |
| `SATA3G_5` and `SATA3G_6` | Empty | `ata5: DUMMY` and `ata6: DUMMY`. The OS does not get them, and they are not tested ([hardware.md](../docs/hardware.md)) |

To map the ports, use `ls -l /dev/disk/by-path/`: `ata-N` is `SATA6G_N`. The `sdX` letters change between boots. ZFS finds the members of the pool by GUID. So the port order is important only when you decide which drive to remove.

### `SATA6G_1`

Two of the four cables do not start a drive on this port, but they operate at 6 Gb/s on ports 2 and 4. The other two cables work on this port. So the port is the fault, probably worn contacts on a board that is 13 years old. All four cables are good.

The burn-in cleared the port on 11 August: attribute `199` was zero after approximately 14 hours of I/O at full rate, with no link resets and 6.0 Gb/s all the time. Keep the cable that works. A looser connector brings back a drive that never appears. To examine the port again, see "Is the SATA link healthy?" in [commands.md](../commands.md).

## The pool `NAS_Storage`

| Item | Value |
| --- | --- |
| Topology | 1 × RAIDZ1 of 3 × 3.64 TiB, with no auxiliary vdevs ([storage.md](../docs/storage.md)) |
| `ashift` | 12. Verified with `zpool get ashift NAS_Storage` immediately after the pool was made. It is fixed for the life of the vdev |
| Encryption | Off. A lost key loses the pool, and no threat here is worth that risk. A child dataset can still be encrypted by itself |
| Usable | 7.28 TiB before the overhead. The plan uses approximately 5.7 TiB |

SMART was read again on all three drives immediately before the pool was made: attributes 5, 197, 198, 187 and 188 were zero, and `183` was still 6 on one drive.

## Datasets

The layout and its reasons are in [storage.md](../docs/storage.md). Make each dataset in *Storage → NAS_Storage → Add Dataset*, with the defaults and the values below.

The **Dataset Preset** sets `casesensitivity`, which can never change. **SMB** means NFSv4 ACLs and no case sensitivity. **Generic** means POSIX ACLs and case sensitivity. Select the preset by this question: will a client that ignores case ever write to the dataset?

| Dataset | Preset | Access Time |
| --- | --- | --- |
| `NAS_Storage/media` | Generic | Off |
| `NAS_Storage/personal` | SMB | Inherit |
| `NAS_Storage/apps` | Generic | Inherit |
| `NAS_Storage/home` | Generic | Inherit |

In `media`, the folders are plain directories, never datasets:

```bash
mkdir -p /mnt/NAS_Storage/media/library/{movies,series,anime}
```

A child dataset below a parent that is bind-mounted can look empty in a container. That is one more reason to use directories.

`personal` uses NFSv4 ACLs, so the POSIX group and the setgid bit below do not apply to it. Accounts for other persons are deferred, because nobody else stores data here now.

## Permissions: the media group

| Item | Value |
| --- | --- |
| Group | `media_access`, GID 3002. It owns the data on `media`. It has no privileges, and *SMB Group* is off, because the dataset is POSIX |
| SMB user | `<SMB_USER>`, UID 3001. Its primary group is 3003 (its own), and its auxiliary groups are `media_access` and `builtin_users`. It is for SMB only: no shell, no SSH and no web UI. Its home is `/var/empty` |
| Containers | `PUID 568` (`apps`), with `PGID 3002` where they need the media group |

The ownership is stored as numbers. So a new build must use the same IDs.

The setgid bit makes the shared group work, and the web UI cannot set it. The editor of the web UI has only the nine `rwx` boxes, and a later save there removes the bit with no message. Set it in a shell, if possible while the directory is empty:

```bash
chmod 2775 /mnt/NAS_Storage/<dataset>     # the result must be drwxrwsr-x
```

- Without setgid, the files of each writer get the primary group of that writer. `2775` is setgid and group write. Both are necessary.
- Through SMB, Samba sets the group itself. But it removes setgid from the directories that it makes, because no `default:` ACL entries exist. Containers that write directly do not get that help. So after each bulk write through SMB, run this command:

  ```bash
  sudo find /mnt/NAS_Storage/<dataset> -type d -exec chmod g+s {} +
  ```

  `sudo` is necessary, because the SMB account owns the directories.
- The group is not a security boundary. The mount of a container is what limits a service. Do not mount all of `/mnt/NAS_Storage` into a container.

## Container mounts

| Container | Host path | Container path | Mode |
| --- | --- | --- | --- |
| Jellyfin | `/mnt/NAS_Storage/media/library` | `/media` | Read-only |

The state of each app is on a host path in `NAS_Storage/apps/<service>`. Each service runbook gives its paths.

## Snapshots

Two periodic snapshot tasks run, both recursive from `NAS_Storage`.

| Item | Daily | Weekly |
| --- | --- | --- |
| Schedule | 00:00 each day | 00:00 each Sunday |
| Lifetime | 2 weeks | 3 months |
| Naming schema | `auto-%Y-%m-%d_%H-%M` | `weekly-%Y-%m-%d_%H-%M` |
| Excluded | `ix-apps` | `ix-apps` and `media` |

- The two name prefixes must be different. The retention applies to each naming schema, so with a shared prefix the daily task would delete the weekly snapshots.
- Do not change the schemas. The replication finds its snapshots by them ([backup.md](../docs/backup.md)).
- A new dataset joins automatically. *Exclude* is the only control. `apps` must stay in both tiers, and `ix-apps` must stay out of both.
- *Exclude* needs the name of the dataset, `NAS_Storage/ix-apps`, not the `.ix-apps` of its mount point. An entry with the name of a dataset that does not exist is accepted with no message, and it excludes nothing. To make sure, run `zfs list -t snapshot -r NAS_Storage/ix-apps`. It must show no `auto-` or `weekly-` snapshots.

### A bulk rewrite of files

A bulk rewrite (a new tag, a remux or a new encode) costs double until the snapshots expire ([backup.md](../docs/backup.md)).

- Examine the free space against the doubled number before the job.
- Delete the snapshots of that period only after the result is verified. Until the external backup exists, they are the only way back.
- Matroska can be edited in place: `mkvpropedit` writes a few KB for each file. A change to MP4 metadata writes the full file again. Divide a job by container before you calculate its size.
- A bulk rewrite is cheapest before the replication exists. After that, the new blocks must also be sent, not only stored.

## Scrub

TrueNAS makes the scrub task with the pool. Its settings since 11 August 2026: each month on the 1st at 00:00, with a threshold of 25 days. Auto TRIM is off, because the drives are HDDs.

- *Threshold days* skips a scheduled run, with no message, if a scrub ran in that number of days. With 30, the run of 1 March is skipped each year, because 1 February to 1 March is 28 days. Keep the threshold below the shortest interval of the schedule.
- The first scrub took 1 second on an empty pool. It proves that the task works, and nothing more.

## SMART on 25.10

This release has no SMART service and no page for tests. `midclt call core.get_methods` shows no SMART methods, and `smartd.service` does not exist. `smartctl` works, and the alerts stay: *Alert Settings → Hardware* still has Failed Selftest, Disk Temperature Is Too Hot and Uncorrected Errors Detected. It is not confirmed yet that the failure of a test that was started by hand causes the alert.

The schedule is planned. It goes in *System Settings → Advanced → Cron Jobs*. The middleware manages those jobs, so they survive updates and are in the config backup. An edit of `/etc/crontab` is not.

```sh
for d in /dev/disk/by-id/ata-ST4000VN006*; do
  case "$d" in *-part*) continue ;; esac
  smartctl -t long "$d"
done
```

- Select the drives by model, never by `sdX`. That also keeps the serials out of the job.
- The planned schedule: a long test each month on the 15th, and a short test each week in the middle of the week. Then the tests do not run with the scrub or with the snapshots of Sunday.
- A scrub and a SMART test do different work. The scrub examines the checksums of the data. The SMART test examines the mechanics of the drive.
- Scrutiny is the option if a dashboard for the SMART history becomes necessary.

## A bulk copy from Windows

Use this procedure for a large copy from a Windows machine to a share. The copy of 12 August 2026 is in the [build log of phase 1](../build-log/phase-1.md).

1. Find the file names that are too long. Linux counts the limit of 255 in bytes, and NTFS counts it in characters. Make the descriptive part of the name shorter, not an ID at the end.

   ```powershell
   $enc = [System.Text.Encoding]::UTF8
   Get-ChildItem -LiteralPath "<source>" -Recurse | Where-Object { $enc.GetByteCount($_.Name) -gt 255 } | Select-Object FullName
   ```

2. Copy with `robocopy`. It can continue after a stop, it writes a log, and it skips a file that fails. Keep the machine awake during the copy.

   ```powershell
   robocopy <source> \\<NAS_IPV4>\<share> /E /J /R:2 /W:5 /NP /TEE /LOG+:C:\copy.log
   ```

3. Verify on the receiving side, with `find … -type f | wc -l` and `du -sb`. Do not use the summary of the copy tool.
4. Set the setgid bit again (see "Permissions: the media group").
5. Remove the share.

- Never use `/MIR` or `/PURGE` from a source that holds only new files. They delete everything in the destination that is not in the source.
- An exit code of `robocopy` below 8 is a success. `1` means that files were copied.
- On a gigabit link, `robocopy` is not faster than Explorer: approximately 3 to 4 hours for each TB.
- A transfer that stays near 11 MB/s is a 100 Mb/s link.
- Keep the source until a backup exists and a restore from it was done ([backup.md](../docs/backup.md)).
