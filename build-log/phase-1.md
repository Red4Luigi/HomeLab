# Build log, phase 1: NAS foundation

This file records what happened in phase 1, with dates: the problems, the wrong assumptions and the measurements. The configuration is in the [TrueNAS runbook](../runbooks/truenas.md), and the decisions are in [`docs/`](../docs/). The log identifies a drive by its port, never by its serial.

## Approximately 2026-07-26: the connector that stopped the first build

**Symptom.** The main power connector of the Medion H81H3-EM2 board fits no retail PSU.

**Cause.** The connector is a proprietary 2 × 7 (14-pin) connector, not the 20-pin or 24-pin ATX standard. No retail PSU can power it. The first PSU also had no SATA power at all. So the board and the PSU depended on each other, and both had to go.

This result changed the project. It started as a new use for an old desktop, and it became a new build. Only the CPU cooler and two SATA cables stayed.

The date is approximate.

## 2026-08-08: assembly

The board, the CPU, the RAM, the PSU and four drives went into the Node 804, with the I/O shield. The case fans went on the 3-speed controller of the case, not on headers of the board. I replaced the CMOS cell. The UPS passed its arrival test: I disconnected the wall plug under a live load, and the transfer to the battery worked.

Nothing broke. The printed I/O shield fitted correctly, and each standoff had its hole in the board. Before the print, I made sure that the plate was for the plain B85M-G and not for the PLUS variant.

Two warnings of the [build checklist](../docs/build-checklist.md) did not become problems. I record them as measurements:

- The A850GL fits in the 160 mm depth limit of the Node 804, with the bottom drive cage full and nothing in its way.
- The fan headers of the board were never a limit. All three case fans are on the controller of the case, and only `CPU_FAN` is in use. The cost: these fans no longer follow the temperature ([hardware.md](../docs/hardware.md)).

### The 4-pin CPU cable that was not missing

**Symptom.** The box of the PSU had no 4-pin ATX12V lead, and the board needs one for the POST.

**Cause.** A modern PSU has an 8-pin EPS lead that is made of two 4-pin halves. The halves come apart. So one cable is correct for boards with a 4-pin socket, such as the B85M-G, and for boards with an 8-pin socket. Divide the lead and connect one half.

**Next time.** It looks the same as a missing cable. Know this before you order one.

### I skipped the bench test on purpose

The build checklist asks for a bench test before the assembly: the CPU, one memory module and the PSU on the bench. I did not do it. All the parts arrived at the same time. A direct assembly made it easier to plan the cables as a whole. The board also did not stay out of the case between steps.

A board that was dead on arrival would show only after the full assembly, and the diagnosis would compete with the return period of the seller. The first POST on 9 August was clean.

## 2026-08-09: from the first power-on to the burn-in

The first POST answered the question of the skipped bench test: the used board was not dead on arrival. I flashed the BIOS, configured it, proved it through a full cycle without mains power, and installed TrueNAS Scale. There were two real problems and one near miss.

### The board still had its firmware of 2013

The used B85M-G reported BIOS 0703, of 2013-10-30. The owner before me never updated it in thirteen years.

I flashed 3602 (2018-03-23), the last release of ASUS for this board.

**Next time.** Flash before you configure. A flash resets the settings to the defaults. In the opposite order, the flash deletes the settings that the test without mains power must verify.

### Secure Boot refused the installer of TrueNAS

**Symptom.** *"The system found unauthorized changes on the firmware, operating system or UEFI drivers."* I had just flashed the BIOS of a used board. So this message looked like a problem with the board, not like a configuration problem.

**Cause.** The BIOS flash turned Secure Boot on again, with a platform key loaded. The installer of TrueNAS is not signed with a key that the board has. The message had no relation to the owner before me.

**Fix.** `Boot → Secure Boot → OS Type → Other OS`.

**Next time.** A BIOS update turns Secure Boot on again. Examine the setting after each flash.

### A drive did not appear on SATA port 1

**Symptom.** The BIOS did not find a drive on `SATA6G_1`.

**Assumption.** In this order:

1. The cable, then the drive. I moved the drive to a 3 Gb/s port with a different cable, and it appeared.
2. From that, I concluded that two of the four cables were bad and could not hold a 6 Gb/s link. This was wrong, and for a short time it was in the plan as a fact.

**Cause.** The port. A test with one variable at a time showed that both "bad" cables ran a drive at 6 Gb/s on ports 2 and 4. The two other cables worked on port 1. Port 1 was the only part that was present in each combination that failed. All four cables are good.

The fault depends on the cable because port 1 is degraded, not dead. The probable cause is worn contacts or a very small crack at the solder pads, on a board that is 13 years old. The shells of the connectors have different tolerances between manufacturers. A cable that holds tighter closes the weak contact, and a looser cable does not.

**Fix.** None was necessary. The board has six ports for four devices. The port stayed on probation, and the burn-in gave the verdict: the negotiated link speed, the link resets in `dmesg`, and SMART attribute 199. The burn-in is the correct time for this test, because the drive is empty. In a pool, a drive that drops from such a port looks the same as a drive that fails.

**Next time.** Change one variable at a time. I changed the cable and the drive together. That made an evening out of a diagnosis of five minutes, and it gave a wrong answer that I was sure of.

### Two `badblocks` processes ran on one drive

**Symptom.** The first start printed `nohup: ignoring input`. It looked like a failure, and I started the command again.

**Cause.** `nohup: ignoring input` is only information. `nohup` prints it each time that stdin is a terminal. The first process had run correctly for one minute. The second start made a second `badblocks` in write mode on the same device.

**Why it was important.** `badblocks -w` writes its pattern on the full disk. Then it reads the disk and expects to find the pattern. Two runs at the same time write over each other, so the run that survives reads the data of the other run. It reports almost each block as bad, which is a catastrophic result on a healthy drive. This happened on `sdb`, the drive on `SATA6G_1`. A false failure there would have looked like a confirmation of the port fault.

**Fix.** I stopped both processes, not one. The disk already had patterns from both runs, so the result of the survivor had no value. I started one run again from block 0.

## 2026-08-10: the night that did not happen

**Symptom.** On the next day, all three `.out` files ended with `Interrupted at block N`. The runs were approximately 1% complete, and all three stopped at the same moment.

**Assumption.** A disk fault or an I/O error stopped the run. `Interrupted at block N` looks the same as such a fault.

**Cause.** The end of the session. `nohup` only makes a process ignore SIGHUP. A logout on a machine with systemd stops the cgroup of the login session with SIGTERM, which `nohup` does not stop. `sudo -i` did not help, because it stays in the same session. `badblocks` handles the signal correctly and prints `Interrupted at block N` when it stops. So the message describes a clean stop, not a fault.

**Fix.** I started each run again as a transient unit of the system manager. The unit is in `system.slice`, where a logout cannot stop it:

```bash
sudo systemd-run --unit=bb-sda --collect /bin/sh -c 'badblocks -wsv -b 4096 -t random -o /tmp/bb-sda.log /dev/sda > /tmp/bb-sda.out 2>&1'
```

**Cost.** Approximately 3.5 minutes of drive time, because `badblocks` always starts again from block 0. It also cost a full night of clock time.

**Next time.** Use `systemd-run` or tmux for each job that must run longer than a session. A SMART long test does not have this risk: `smartctl -t long` gives the work to the firmware of the drive. The test survives a logout and a reboot.

### Three messages that look like failures and are not

All three cost time in one run.

| Message | Meaning |
| --- | --- |
| `nohup: ignoring input` | Information. Each `nohup` log starts with it |
| `Interrupted at block N` | `badblocks` got a signal and stopped cleanly. It is never a bad disk |
| A progress line that never appears | The buffer of stdio. When the output goes to a file, glibc changes from a line buffer to a full buffer. One drive showed nothing for 17 minutes, and it wrote faster than the other two |

One lesson repairs all three: verify from the block layer, not from the output of the program. Field 7 of `/sys/block/<dev>/stat` is the number of written sectors. SMART attribute `241 Total_LBAs_Written`, divided by the total number of sectors, gives the progress. `systemctl status` shows that the CPU time increases. A blocked stdout cannot make these wrong.

## 2026-08-11: the burn-in passed, and SATA port 1 with it

The burn-in was the last gate of phase 1 before the pool. Both halves ran: the destructive write, read and compare of the full surface, and the SMART long test of each drive.

`badblocks -wsv -b 4096 -t random` completed on all three drives: `Pass completed, 0 bad blocks found. (0/0/0 errors)`. That is a full destructive write and a full read and compare of 4 TB for each drive, in approximately 14 hours.

SMART against the baseline of 9 August: each attribute that decides a return is still zero on all three drives (5, 197, 198, 187 and 188). No drive had to go back.

**`SATA6G_1` is cleared.** `199 UDMA_CRC_Error_Count` is zero on the drive on the degraded port, after fourteen hours of continuous I/O at full rate. `dmesg` recorded no link reset in that time. The result is narrower than "the port is good". The port is weak at link training, against the tolerances of some connectors. It is solid when a link exists with a cable that fits correctly. The diagnosis of a mechanical contact predicted this. So the rule is not "do not use port 1". The rule is "do not change the cable of port 1 without a reason". A cable with a looser shell brings the fault back.

**`183 Runtime_Bad_Block` stayed at 6** on the one drive that reports it, through 38 hours of power and 4 TB written and 4 TB read. The value does not increase, so it comes from the factory and is not a defect. Without the baseline of 9 August, a `6` that I found three months later would have no explanation.

**Two numbers that look bad and are not.** The normalised value of `1 Raw_Read_Error_Rate` went from 100 to 76–83 (worst 69–70), and `7 Seek_Error_Rate` went to 66. Both are combined counters of Seagate for errors and operations. The sign: attributes 1 and 195 have the same raw value on each drive, which a count of errors would not have. The thresholds are 6 and 45, so the margins are very large.

**The SMART long tests also passed.** They ended at 18:13 with `Extended offline · Completed without error` on all three drives. `LBA_of_first_error` is empty, so no sector was unreadable. The column `Remaining` shows `00%` on a completed test. That looks like a failure, but it means that zero percent is left. `LifeTime(hours) 47` agrees with the run: 40 hours after `badblocks`, plus the test of approximately 7 hours.

**The counters agree with the logs.** `242 Total_LBAs_Read` was approximately 7,814,098,521, and the drive has 7,814,037,168 sectors. So the read pass really read each sector. `241 Total_LBAs_Written` is approximately 76 million sectors (approximately 39 GB) above one full drive. That is almost exactly the 0.97% that the first run reached before the logout stopped it.

## 2026-08-11: the password prompt that came back

**Symptom.** I set *Allowed Sudo Commands (No Password)* to `ALL` for my account. `sudo` still asked for the password, also in a new session.

**Assumption.** The setting was not saved, or the session had to start again.

**Cause.** Both sudo fields had `ALL`. That makes two rules: `ALL=(ALL) ALL` and `ALL=(ALL) NOPASSWD: ALL`. In sudoers, the last match wins, and the plain rule asks for the password. The web UI still showed both fields as set.

**Fix.** I cleared *Allowed sudo commands* and kept only the field without password ([TrueNAS runbook](../runbooks/truenas.md)).

**Next time.** Remove the cached timestamp before each test ([commands](../commands.md)). sudo keeps it for approximately 15 minutes. So each check after the first password tested the cache, not the rule.

## 2026-08-12: the copy from Windows

I copied approximately 1 TB from the Windows workstation to the NAS, through a temporary SMB share, before the workstation moved to Linux. An SMB share can be on a Generic dataset, because Samba handles the lookups that ignore case. So a temporary SMB dataset was not necessary.

| Item | Value |
| --- | --- |
| Transferred | 2209 files in 414 directories, 1,104,361,709,981 bytes (1028.5 GiB) |
| Time | 2:04:21 at approximately 111.5 MB/s. The gigabit link was full |
| Failures | 0. I made two file names shorter before the copy |

Two facts from this copy:

- **Linux counts the limit of 255 for a file name in bytes. NTFS counts it in characters.** A CJK character uses 3 bytes, and an emoji uses 4. So a name of 118 characters can be 278 bytes, and the copy of that file fails. Copy tools record the failure and continue, so files are missing with no clear sign. Find the long names before the copy.
- **A transfer that stays near 11 MB/s is a 100 Mb/s link.** That was the first attempt here.

## 2026-08-26: the migration, and two things that were not what they looked like

This was the move of the workstation from Windows to Linux. I moved the data off the old installation and formatted three internal drives with btrfs. The rule of the [backup](../docs/backup.md) applied through all of it: format the source last.

### The format that never ran

**Symptom.** I selected the 1.8 TB games drive in KDE Partition Manager, selected btrfs and clicked OK. After that, the drive mounted again and showed each file as before.

**Assumption.** The format ran and failed with no message. Or, worse, it ran on the wrong target.

**Cause.** Neither. The format never started. Two independent locks stopped it. KDE Partition Manager puts operations in a queue and writes nothing until you click Apply. It also refuses to change a file system that is mounted, and that drive was mounted. `lsblk` confirmed it: still `ntfs`, with the same label and the same UUID. The two other drives were the same.

**Fix.** Unmount first, then click Apply. Nothing was lost, because nothing had happened.

**Next time.** Nothing different. The check of the result found it.

### The transfer that was not slow

**Symptom.** A copy of a large `AppData` tree from the NAS through `smb://` in Dolphin was very slow: hours for tens of GB, on a gigabit LAN.

**Assumption.** The network, or a NAS under load.

**Cause.** The protocol layer. `smb://` in Dolphin is a KIO worker, not a mounted file system. It keeps directory listings in a cache and makes some round trips for each file. On a tree of hundreds of thousands of small files, the latency decides everything, and the bandwidth is not important. The old listings also caused "file does not exist" errors during bulk deletes: the files were already gone, but the cached listing still showed them.

**Fix.** rsync through SSH. I measured 927 Mb/s on a 1000 Mb/s link, so the transfer was never the limit. A CIFS mount of the kernel is the middle option if SMB is necessary. Do not use `smb://` for bulk work.

**Next time.** When a transfer is slow, measure the link before you adjust anything. Four seconds of `/proc/net/dev` showed the difference between "the network is slow" and "the protocol is wrong".

## Measured

| Item | Value |
| --- | --- |
| BIOS as bought | 0703 (2013-10-30). Flashed to 3602 (2018-03-23) |
| RAM available | 15,667,288K of 16,646,600K. 16 GB confirmed |
| SATA links | All four negotiated 6.0 Gb/s, port 1 included |
| SMART baseline | `Power_On_Hours` 2 and `Power_Cycle_Count` 7 on all three drives. Really new stock from both sellers |
| Return attributes at the baseline | 5, 197, 198, 187, 188 and 199: all zero on all three drives |
| Write rate | Approximately 187 to 196 MB/s for each drive, three drives in parallel. As the IronWolf specification |
| Drive temperatures at idle | 37 to 39 °C |
| Drive temperatures, peak during the burn-in | 42 °C (39 / 42 / 42), with the fans on the controller of the case at high. The limit for action is 50 °C |
| `badblocks` runtime, one random pattern | Approximately 14 hours for 4 TB: 7 hours of write and 7 hours of read and compare, at an average of approximately 157 MB/s |
| Burn-in result | `0 bad blocks found. (0/0/0 errors)` on all three drives |
| SMART long test | `Completed without error` on all three drives, with no `LBA_of_first_error` |
| Return attributes after the burn-in | 5, 197, 198, 187 and 188: still zero on all three drives |
| `199` on the drive on `SATA6G_1` | 0 after approximately 14 hours of I/O at full rate. The port passes |
| LAN rate, NAS to PC with rsync through SSH | 927 Mb/s of a 1000 Mb/s link. Line rate, small files included |
| The same tree through `smb://` in Dolphin (KIO) | Fewer than 10 MB/s. The protocol is the limit, not the bandwidth |
| Volume of the migration | Approximately 90 GB |

The plan expected a possible thermal problem in this case. Under the heaviest continuous load that the build will have, there is none: the margin is eight degrees.

The first estimate of approximately 12 hours for this run was wrong. It came from the first minutes, when the heads are on the outer tracks at approximately 190 MB/s. The sequential rate decreases toward the spindle, so the average of the full disk is near 157 MB/s. Estimate a pass of the full surface from approximately 150 MB/s, not from the rate at the start.
