# Hardware

This file describes the machines of the homelab, the parts of the NAS, the case and the rack, with the reason for each decision. The storage is in [storage.md](storage.md), and the power protection is in [power.md](power.md).

## The machines

| Machine | Specification | Role | Always on |
| --- | --- | --- | --- |
| The NAS | i3-4130, 2 × 8 GB DDR3-1600, ASUS B85M-G, MSI MAG A850GL, 3 × IronWolf 4 TB, Intenso 128 GB boot SSD, no discrete GPU | NAS, Jellyfin, containers and public web hosting | Yes |
| Intel R1304GL4DS9 | 1U server with two Xeons and IPMI. Loud, approximately 80 to 120 W at idle | LFS, then game-server hosting. These projects have their own repositories | No. IPMI starts it when it is necessary |
| Raspberry Pi 3 Model B+ | Cortex-A53, 1 GB of RAM. Its gigabit NIC is on a USB 2.0 bus (approximately 300 Mb/s). It cannot transcode | Pi-hole and NUT client | Yes |
| TP-Link TL-SG108E | Switch with 8 gigabit ports. It can do VLANs. Approximately 100 mm deep | The switch of the homelab | Yes |
| MacBook Pro | Used. The model and the year are not recorded yet. Erased | None | No |

The spares of the NAS are a G3240 CPU, a 4 GB memory module and the old HDD.

## The parts of the NAS

### Board: ASUS B85M-G

mATX, B85 chipset, LGA1150, 4 × DDR3 (32 GB maximum), 4 × SATA 6 Gb/s and 2 × SATA 3 Gb/s, 24-pin ATX, FAN Xpert, no M.2. The last BIOS is 3602, from 2018. The board was bought used for €46, with the i3-4130 included.

The first plan was to use the board and the PSU of the old PC again: a Medion H81H3-EM2 and an FSP350. Both were replaced, for these reasons:

- The main power connector of the Medion board is a proprietary 2 × 7 (14-pin) connector, not ATX. No retail PSU can power it.
- The FSP350 has no SATA power at all. The old HDD ran from a Molex adapter. The optical drive ran from a header on the board, which sends drive power through the traces of the board.
- So to keep one of the two parts was to keep both.

The replacement of both parts cost approximately €120. It also solved the fit of the board, because each retail mATX board fits the Node 804.

Other boards were considered and rejected, for these reasons:

- An unknown brand, or no reviews
- A late delivery
- The ATX size, which the Node 804 cannot hold
- Only three SATA ports. That forces a boot from NVMe, which the 8-series chipsets do not support without a modified BIOS

The B85M-G has six SATA ports.

### The two SATA 3 Gb/s ports

It is not known if the two SATA 3 Gb/s ports work. In `dmesg`, `ata5` and `ata6` show `DUMMY`: the firmware never gives them to the OS. An empty port that works shows `link down`.

These ports are the assumed spare capacity if the degraded port 1 is retired ([build-checklist.md](build-checklist.md)). The planned test: connect the old spare HDD to `SATA3G_5` and see if the OS finds it.

### CPU: i3-4130

2 cores and 4 threads, 3.4 GHz, HD 4400 graphics (20 EU), 54 W. The RAM runs at its rated 1600.

The spare is a G3240: 2 cores and 2 threads, 3.1 GHz, 10 EU. It limits the RAM to 1333.

Neither CPU has hardware for HEVC. That is a limit of the Haswell generation ([services.md](services.md)).

### PSU: MSI MAG A850GL

850 W, 80+ Gold, fully modular, ATX 3.0, with a 120 mm FDB fan. It has 2 SATA leads and 1 Molex lead. A PSU for the Node 804 must be 160 mm deep or less.

### Drives: ST4000VN006

CMR, 5400 rpm class, approximately 4 to 5 W at idle, 512e.

## Case and cooling

### Case: Fractal Node 804

An mATX cube with 8 bays for 3.5" drives. It has two chambers, so the drives are apart from the board. It includes fans, and it is quiet. It is approximately 30 cm tall, which is approximately 7U.

Other cases were considered and rejected, for these reasons:

- **Too few bays for 3.5" drives.** Gaming cases no longer have 3.5" cages, and specification tables count 2.5" mounts as bays. Read the `3.5"` line of the manufacturer. A cage cannot be added to a case that has no mounting points for it.
- **Too tall for the rack.** A tower with three bays is 52 cm tall (approximately 12U), and it cannot hold more than three drives. The Node 804 has 8 bays in approximately 7U.
- **A 2U rackmount chassis.** It needs a PSU that vents to the rear and is 14 cm deep or less. An ATX PSU with the fan at the bottom does not fit.
- **A mini PC or an SFF desktop.** These have no support for 3.5" drives.

### No discrete GPU

Quick Sync runs on the iGPU. A card would only add heat and power draw.

### Cooling

- **CPU cooler:** the cooler of the old PC, with new thermal paste. The OEM air duct is removed. Its fan is the part that wears, because it runs 24 hours each day.
- **Case fans:** they run on the 3-speed controller of the Node 804, which a SATA lead of the PSU powers. So they hold a fixed speed, and they do not follow the temperature of the drives.
- **CPU fan:** it is on `CPU_FAN`, with PWM control.

### Temperatures

TrueNAS sends the alert *Disk Temperature Is Too Hot* through Telegram.

| Drive temperature | Meaning |
| --- | --- |
| Below 40 °C | Comfortable |
| Low 40s | Acceptable |
| Above 50 °C for a long time | Do something |

Measured: a peak of 42 °C during approximately 14 hours of burn-in, and a constant 43 °C during a full scan of the library.

A scrub is not the worst case, because it reads only allocated data. The next real thermal load will be a resilver.

## The R1304

- **Specification:** two Xeon E5-2600 or v2 CPUs, 1U, 710 mm deep, approximately 9.5 kg, IPMI, four gigabit NICs, DDR3, one 460 W PSU.
- **Drives:** 4 hot-swap bays for 3.5" drives, with 4 × 300 GB drives. Their interface and their age are not recorded.
- **Noise and power:** loud, approximately 80 to 120 W at idle.
- **Remote console:** the BMC alone gives power control, sensors and Serial-over-LAN. A graphical remote console needs the RMM4 Lite key module of Intel. It is not verified if this unit has one. The console of that generation is a Java applet.
- **Cables:** the four NICs need only one or two cables: one for the BMC and one for the OS.

The steps to protect the BMC are in [security.md](security.md).

## Rack and placement

The rack is a StarTech 15U open frame with 4 posts, 101 cm deep. The server sets the depth of the rack (710 mm), and the tower sets the length of the shelf. The rack was assembled with the depth set for the R1304. To change the depth, each device must come off.

Two Tecmojo 1U vented 4-post shelves hold the machines: one of 550 mm and one of 700 mm. A shelf costs approximately 1U, plus the height of the device on it.

| Device | Place |
| --- | --- |
| R1304 | On the 700 mm shelf, near the front |
| NAS | Upright on the 550 mm shelf |
| UPS | On the floor |
| Pi and switch | Next to the NAS, with the ports to the outside |

The NAS (7U), its shelf (1U) and the server (2U) use approximately 10U of 15U. The NAS fits with approximately 160 mm of margin in the depth.

The reasons for the placement:

- **The tower is upright, not on its side.** On its side, it would save rack units. Upright keeps the airflow by convection as the case is designed. Then the direction of the bays is not important.
- **The NAS is not on the floor of the frame.** That would save a shelf and rack units. But both shelves were already bought, and the build uses approximately 10U of 15U. So no benefit was left to balance the dust and the kicks at floor level.
- **The R1304 has no rail kit.** The Intel AXX1U rails give only slide-out access. That has little value on a machine that is off most of the time, is managed through IPMI and is rarely opened.

## The MacBook Pro

The MacBook Pro has no role. Its model and its year decide if it can run Linux: an Intel model is easy, and Apple Silicon has narrow support.
