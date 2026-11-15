# Commands

This file helps to find the correct command when something goes wrong. The commands are in groups, by the question that they answer. This file has the commands. The reasons are in [`docs/`](docs/) and in the [runbooks](runbooks/).

The placeholders are `/dev/sdX` for a disk, and `<NAS_IPV4>` and `<user>` for the NAS. The real values are not in this repository. "Not run yet" marks a command that was not run on this build.

On TrueNAS, change state through the web UI or the middleware. TrueNAS makes its system files again from its own database, so an edit by hand in `/etc` can be lost after an update. Commands that only read are always safe. The commands that change state have a mark.

## Is this drive healthy?

```bash
sudo smartctl -i /dev/sdX          # identity: model, firmware, 512e or 4Kn
sudo smartctl -H /dev/sdX          # the general verdict. Necessary, but not sufficient
sudo smartctl -A /dev/sdX          # the attribute table. This one decides
sudo smartctl -x /dev/sdX          # everything, with the error log and the temperature log
```

- Return a new drive if one of these is not zero: 5 `Reallocated_Sector_Ct`, 197 `Current_Pending_Sector` or 198 `Offline_Uncorrectable`. Also record 187 and 188.
- 199 `UDMA_CRC_Error_Count` counts link errors: the cable or the port, not the media.
- 183 is specific to the vendor. A value that does not change comes from the factory. A value that increases is a defect. Only a baseline shows which one it is, so record one when a drive starts for the first time ([build checklist](docs/build-checklist.md)).

## How do I run a self-test on a drive?

```bash
sudo smartctl -c /dev/sdX               # how long the tests take
sudo smartctl -t short /dev/sdX         # approximately 2 minutes
sudo smartctl -t long /dev/sdX          # hours
sudo smartctl -l selftest /dev/sdX      # results. "00%" remaining means complete
sudo smartctl -X /dev/sdX               # stop the test
```

All the pool disks, by their stable name. This also keeps the serials out of the text that you paste:

```bash
for d in /dev/disk/by-id/ata-ST4000VN006*; do
  case "$d" in *-part*) continue ;; esac      # by-id also shows partitions
  sudo smartctl -t long "$d"
done
```

The test runs in the firmware of the drive, so it survives a logout and a reboot. TrueNAS 25.10 schedules no tests. Use *System Settings → Advanced → Cron Jobs* ([TrueNAS storage runbook](runbooks/truenas-storage.md)).

## How do I test the full surface of a new drive?

This test destroys all data on the target. Use it only before the drive is in a pool.

```bash
sudo systemd-run --unit=bb-sda --collect /bin/sh -c \
  'badblocks -wsv -b 4096 -t random -o /tmp/bb-sda.log /dev/sda > /tmp/bb-sda.out 2>&1'
systemctl status bb-sda
```

- Use `-b 4096` on large drives. The default of 1024 bytes failed here with "Value too large for defined data type".
- The command needs root. When the output goes to a file, a permission failure shows no message.
- `-t random` is one pattern, approximately 14 hours for 4 TB. The default of four patterns takes approximately 50 hours.
- Use `systemd-run`, not `nohup`. A logout stops the session with SIGTERM, which `nohup` does not stop ([build checklist](docs/build-checklist.md)).
- Never run two `badblocks -w` on one disk. Both then report each block as bad. Stop both, and start one again.

## Does that long job really do something?

Examine a layer that the program cannot make wrong:

```bash
cat /sys/block/sda/stat                                # field 7 = sectors written (× 512 bytes). Read it two times, 10 seconds apart
sudo smartctl -A /dev/sda | grep -E "241|Total_LBAs"   # progress against the total number of sectors
systemctl status bb-sda                                # the CPU time increases
```

No output is not evidence of a stop. The stdout that goes to a file has a buffer.

## Which physical disk is this?

```bash
ls -l /dev/disk/by-path/         # ata-N is SATA6G_N on this board
ls -l /dev/disk/by-id/           # model and serial
lsblk -o NAME,SIZE,MODEL,SERIAL,MOUNTPOINT,TYPE
```

The `sdX` letters change between boots. Connect each result to a serial.

## Is the SATA link healthy?

```bash
dmesg | grep -i "SATA link"      # expect 6.0 Gbps
dmesg -T | grep -i ata           # resets, failed commands, DUMMY ports
sudo smartctl -A /dev/sdX        # attribute 199
```

The known state of this board: `SATA6G_1` is weak but cleared, and `ata5` and `ata6` report `DUMMY` ([TrueNAS storage runbook](runbooks/truenas-storage.md)).

## What state is the pool in?

```bash
zpool status -v                  # health, errors, progress of a scrub or a resilver, damaged files
zpool list -v                    # capacity and fragmentation, for each vdev and disk
zfs list                         # datasets
zfs list -t snapshot
zpool iostat 5                   # live rate
zpool history NAS_Storage        # each admin change, with the commands of the web UI
```

Most `zpool` and `zfs` commands that only read work without `sudo`.

## What are the real settings of the pool?

```bash
zpool get ashift NAS_Storage     # 12. Fixed when the pool was made
zfs get casesensitivity,compression,recordsize,acltype NAS_Storage/<dataset>
zfs get all NAS_Storage/<dataset>
```

To change `ashift` or `casesensitivity`, you must destroy the pool or the dataset and make it again ([storage.md](docs/storage.md)).

## How do I scrub the pool?

These commands change state. They were not run yet on this build.

```bash
sudo zpool scrub NAS_Storage         # start
sudo zpool scrub -p NAS_Storage      # pause
sudo zpool scrub -s NAS_Storage      # stop
```

Schedule the scrub as a TrueNAS task. Use the CLI to monitor it.

## Why does sudo ask for a password?

```bash
sudo -k; sudo true     # remove the cached timestamp first. If not, you test the cache
sudo -l                # the rules that apply, in their order
id; getent passwd <user>
```

In sudoers, the last match wins. With `ALL` in both sudo fields of TrueNAS, the plain rule wins against `NOPASSWD` ([TrueNAS runbook](runbooks/truenas.md)). sudo reads sudoers again at each call. It reads the group membership only at a new login.

## SSH keys and access

```bash
ssh-keygen -p -f ~/.ssh/id_ed25519          # add or change the passphrase. The public key stays the same
ssh-keygen -lf ~/.ssh/id_ed25519.pub        # fingerprint
ssh-add ~/.ssh/id_ed25519                   # one time in each session, into the agent
ssh-add -l                                  # what the agent holds
ssh -v <user>@<NAS_IPV4>                    # why the authentication failed. Read from the bottom up
```

On a Windows laptop, the agent is a service, and its default start type is `Manual`:

```powershell
Set-Service ssh-agent -StartupType Automatic     # as administrator
Start-Service ssh-agent
ssh-add $env:USERPROFILE\.ssh\id_ed25519
```

Prove that the key works before you disable the password authentication. If not, the repair needs a monitor and a keyboard.

## What does systemd do, and what went wrong?

```bash
systemctl status <unit>
systemctl list-units --failed
journalctl -u <unit> -f
journalctl -b -p err                 # this boot, errors and worse
journalctl -k                        # the kernel log with history. dmesg has only this boot
```

## Why can this container not write to its host path?

```bash
ls -ln <host-path>                   # owner and group as numbers
stat -c '%a %U:%G %n' <host-path>
docker logs <container> 2>&1 | tail
```

Use `ls -ln`, because the UID of a container frequently has no name on the host. Then find the class of the UID and GID of the container: owner, group or other. Then see if that class can write.

- **Other** (for example a directory that was made by hand as `root:root 755`): use `chown` to the UID of the service. `chmod` cannot help.
- **Group, but no write permission** (for example `root:media_access 2755`, made through SSH): use `chmod g+w` or `chmod 2775`. Never use `775`, which clears setgid. Then the group of new files becomes wrong with time.
- For one writer, make that UID the owner. For more than one writer, use a shared group and setgid.
- For a catalogue app, use the *Enable ACL* entry of the host path. The middleware does not see an owner that was set by hand. For an app from plain YAML, the CLI is the correct way.

## Why do clients still get the old answer after I changed a DNS record?

```bash
resolvectl query <name>              # what the local stub resolver has
resolvectl flush-caches              # also removes negative answers
dig A    <name> @<resolver-ip>       # ask the authority directly
dig AAAA <name> @<resolver-ip>
```

- `Server: 127.0.0.53` in the output of `nslookup` is the cache of your own machine, not Pi-hole.
- NXDOMAIN stays in a cache like each other answer. If a client asks for a name before its record exists, the name stays "missing" for the negative TTL.
- Compare `A` and `AAAA`. With `aa`, Pi-hole answered from its records. Without `aa`, it forwarded the query. A forwarded `AAAA` query that returns NXDOMAIN denies the full name ([network.md](docs/network.md)).

## How do I use a tool that the NAS does not have, one time?

```bash
docker run --rm -it -v /mnt/NAS_Storage/<dataset>:/work debian:bookworm-slim bash
# in the container:
export LC_ALL=C.UTF-8                # slim images use ASCII by default. MKVToolNix then refuses names that are not ASCII
apt-get update && apt-get install -y --no-install-recommends <packages>
```

Never install a tool on the TrueNAS host. The root file system is read-only, and an update replaces it. Run the tool on the NAS, not through SMB. Start a long job detached:

```bash
docker run -d --name <job> -v … debian:bookworm-slim bash -c '<install> && <command>'
docker logs -f <job>                 # Ctrl-C stops only the log view
docker rm <job>                      # -d has no --rm
```

## Why does `sudo cmd > file` say "permission denied"?

```bash
<command> | sudo tee <file> > /dev/null
sudo sh -c '<command> > <file>'
```

The shell opens the target file as your user, before `sudo` runs. A file that `sudo tee` makes has root as its owner. Use `chown` if a container must write to it later.

## Which boot entries exist?

```bash
sudo efibootmgr -v                   # delete the entries that are not yours
```
