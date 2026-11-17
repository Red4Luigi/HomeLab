# Build checklist

This file lists what to examine before a step of a build: for the next drive, the next used board or the next teardown. The events of this build are in the [build log of phase 1](../build-log/phase-1.md). The BIOS settings of the NAS are in the [TrueNAS runbook](../runbooks/truenas.md).

## Before you open the case or install a used board

- **SATA data cables.** Use one cable for each device, from point to point. Bare drives and used boards come without cables. Keep one or two spare cables for swap tests.
- **The CMOS battery of a used board.** Replace it. Do not test it. An empty cell loses the BIOS settings (AHCI, boot order, power-on after AC loss) only after a full loss of power. That is the moment when a headless NAS needs them. With mains power connected, +5VSB powers the CMOS, so a dead cell looks healthy. Keep the old cell as a spare for swap tests.
- **A bench test of a used board.** Test the board with the CPU, one memory module and the PSU before you install anything. A dead board that you find after the full assembly competes with the return period. This build skipped the test on purpose ([build log of phase 1](../build-log/phase-1.md)).
- **The ATX12V plug (4-pin, CPU).** Without it, the board gives no POST and also no beeps. A modern PSU has an 8-pin EPS lead that divides into two 4-pin halves. There is no separate 4-pin cable.
- **Modular PSU cables.** Never use the modular cables of one PSU on a different PSU. A wrong pinout puts 12 V on the 5 V line and destroys the drives. Keep the cables of the A850GL in a bag with the PSU.
- **Adapters for drive power.** If an adapter becomes necessary, do not use a moulded Molex-to-SATA adapter. Its contacts are not crimped, and they can become hot and arc. A crimped SATA-to-SATA Y-splitter is acceptable, up to approximately 4.5 A for each rail.
- **The depth of the PSU.** It must be 160 mm or less for the Node 804.
- **The I/O shield.** Install it before the board, from the inside. Each shield fits each case, because the frame is a standard 158.75 × 44.45 mm. But the cutouts are specific to the board, and a *PLUS* variant is a different board. For a printed shield, use PETG or ASA, because PLA deforms slowly under the tension of the clips. If the board resists at the screw holes and not at the ports, a standoff has no hole in the board.
- **The push-pins of the cooler** (LGA115x, 75 × 75 mm). Before you install the cooler again, turn each pin against its arrow to reset it. Push the pins in diagonal pairs. Old plastic becomes brittle.

### The degraded SATA port

Port `SATA6G_1` of this board is degraded. It is in service, but do not change its cable.

- Two of the four cables do not start a drive on this port. The other two cables operate correctly. The burn-in found zero link errors (SMART attribute 199) in approximately 14 hours at 6 Gb/s.
- The port fails at link training when the connector is looser. The symptom is a drive that never appears.
- If the port ever loses a drive, examine the connector before the disk.
- The fallback ports are not tested yet ([hardware.md](hardware.md)).

Test a port that you do not trust during the burn-in, while the drive is empty. In a pool, an intermittent link looks the same as a drive that fails.

## BIOS, before the TrueNAS installation

Have a USB stick of 8 GB or more for the installer, and a monitor and a keyboard.

1. Flash the BIOS of a used board from the vendor image first. Then configure it. A flash resets each setting, and it gives a clean start on a board with an unknown history.
2. Set the SATA mode to AHCI. ZFS needs direct access to the disks.
3. Set `Restore AC Power Loss` to `Power On`, and `Wait For 'F1' If Error` to `Disabled`. If not, the board waits at a prompt after a power failure.
4. Set `Aggressive LPM` to `Disabled`. Links that wake from idle cause false resets, which look like hardware that fails.
5. Keep the C-states on. They are in *CPU Configuration → CPU Power Management*, one submenu down. Boot with UEFI.
6. Set Secure Boot to `OS Type = Other OS`. See "Secure Boot" below.
7. Prove the CMOS cell. See "How to prove the CMOS cell" below.
8. After the installation, run `efibootmgr -v`. Delete the boot entries that are not yours.

### Secure Boot

Secure Boot refuses the installer of TrueNAS. The message is about "unauthorized changes", and it does not show a compromise. A BIOS flash turns Secure Boot on again, so examine the setting after each flash.

### How to prove the CMOS cell

1. Set all the BIOS settings.
2. Disconnect the power at the machine. With a UPS, the wall plug proves nothing.
3. Push the power button to drain the board.
4. Wait a few minutes.
5. Connect the power again.

If the board starts by itself, the setting and the battery both work.

## New drives, before a pool

- Do a burn-in of each new drive before it goes into a pool. Run a SMART long test and, if possible, a full write and read with `badblocks`. Test the drives in parallel.
- Record the SMART baseline first. `Power_On_Hours` and `Power_Cycle_Count` must be near zero. If they are not, the drive was sold before, and the return period is the time to act. A baseline also gives the later values their meaning. Attribute 183 of one drive here was 6 from the start, and it stayed 6.
- Return a new drive if attribute 5, 197 or 198 is not zero. Also record 187 and 188. Read the attributes again after the tests.
- The burn-in is the real thermal test. Monitor the temperatures. With case fans at a fixed speed, set the switch to high first.

### `badblocks`

The command is in [commands.md](../commands.md).

- Use `-b 4096` on large drives, and run it as root.
- Use one pattern: `-t random`. For 4 TB, it takes approximately 14 hours. Calculate the time from approximately 150 MB/s, not from the rate at the start.
- Start it with `systemd-run`, not with `nohup`. A logout sends SIGTERM to the processes of the session, and `nohup` does not stop that signal. A SMART long test runs in the firmware of the drive, so it survives a logout.
- Read the progress from the block layer: `/sys/block/<dev>/stat` and SMART attribute 241. Do not read it from the output of the program.
- Never start a destructive run again because the output is silent. Two `badblocks -w` runs on one disk report each block as bad.

These messages look like failures, but they are not:

- `nohup: ignoring input`
- `Interrupted at block N`. This is a clean exit after a signal.
- No progress output. The cause is the buffer of stdout.

## When a delivery arrives

- Test a UPS in the return period. Charge it for approximately 8 hours. Connect a small load, disconnect the wall plug, and make sure that the load stays on. A phone charger proves the transfer, not the runtime. The runtime comes from the LCD of the UPS when the real load is connected.
- Examine cases and drives for transport damage before you install them. A claim against the courier is easiest at that time.
