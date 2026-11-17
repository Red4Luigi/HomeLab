# Power

This file describes the PSU of the NAS and the UPS of the homelab, with the reason for each decision. The specifications of the PSU are in [hardware.md](hardware.md).

## The PSU of the NAS

The PSU is an MSI MAG A850GL: 850 W, 80+ Gold, fully modular, ATX 3.0. It replaced the FSP350 of the old PC, which had no SATA power at all.

- **Fully modular.** This is good for a case with two chambers: only the necessary cables are installed, and they go through the divider.
- **As built.** SATA lead 1 powers all three HDDs, and one connector on it is free. SATA lead 2 powers the boot SSD and the fan controller. There are no splitters and no adapters.
- **Too large on purpose.** The idle draw of approximately 45 to 55 W is approximately 6% of the load. The 80+ Gold rating starts at 20%. The lost efficiency costs approximately €5 to €13 each year. This is accepted, because the PSU was cheaper than the 650 W model, it is quiet, and it never gets stress. A measurement with a smart plug is planned.

The rules for modular cables and for adapters are in the [build checklist](build-checklist.md).

The power management of the BIOS is on. The drives do not spin down: the wear of spin-down cycles is not worth it on a machine that runs 24 hours each day.

## The UPS

The UPS is a CyberPower CP900EPFCLCD. It is line-interactive and gives a pure sine wave, which is safe for PSUs with active PFC. Its arrival test is done ([build-checklist.md](build-checklist.md)). It is installed in the rack, and the NAS, the Pi and the switch are on it. The USB cable to the NAS and the NUT configuration are planned.

- **Size.** The UPS is for the load that is always on: the NAS, the Pi and the switch, approximately 55 W. Its purpose is a clean shutdown (approximately 5 to 15 minutes), not operation through a power failure.
- **The R1304 stays off the UPS.** It is the largest load, it has the highest peaks, and it is not always on.
- **No simulated sine wave.** Units with a simulated or step sine wave (for example the Eaton 3S and 5S) are wrong for PSUs with active PFC. A pure sine wave starts at the Eaton 5SC and the CyberPower PFC Sinewave series.

### The NAS is the NUT master

The NAS is the NUT master and has the USB cable of the UPS. The Pi is a client, and it shuts down first.

- The NAS holds the pool, so its shutdown is the important one. That shutdown must not depend on the SD card of the Pi.
- TrueNAS has a built-in UPS service. Its alerts go through the Telegram path that already exists ([services.md](services.md)). On the Pi, NUT would be a setup by hand, with no alerts.

The Pi as the master was the first assumption. The argument for the Pi is that it is the light machine that is always on. Pi-hole runs on the Pi because DNS must not depend on the machine that reboots most. But that reason is about DNS, not about the machine that has the USB cable.

The accepted cost: if the NAS stops, nothing reports the state of the UPS.
