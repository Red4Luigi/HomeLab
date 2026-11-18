# Runbook: TrueNAS Scale, the host

This file records the configuration of the NAS host, sufficient to build the machine again. The pool, the datasets and the data protection are in the [TrueNAS storage runbook](truenas-storage.md). The reasons for the hardware are in [hardware.md](../docs/hardware.md). The events of the build are in the [build log of phase 1](../build-log/phase-1.md).

TrueNAS was installed on 9 August 2026.

| Placeholder | Meaning |
| --- | --- |
| `<NAS_IPV4>` | The fixed IPv4 address of the NAS |
| `<NAS_ULA>` | The static IPv6 ULA of the NAS |
| `<NAS_HOSTNAME>` | The host name of the NAS |

The real values are not in this repository. Drive serials are never in this repository.

## Host

| Item | Value |
| --- | --- |
| OS | TrueNAS-25.10.5. Release 25.10 removed the schedule for SMART tests and `smartd` ([storage runbook](truenas-storage.md), "SMART on 25.10") |
| Board | ASUS B85M-G with BIOS 3602 (2018-03-23, the last release). It arrived with 0703 (2013) and was flashed before the configuration |
| CPU and RAM | i3-4130, 2 × 8 GB DDR3-1600 non-ECC, two slots free |
| PSU and case | MSI MAG A850GL, Fractal Node 804. The case fans are on the controller of the case, at a fixed speed |
| Boot | Intenso 128 GB SSD |
| Pool drives | 3 × Seagate IronWolf ST4000VN006-3CW104, 4 TB, 512e |

The case fans do not follow the temperature. If the drives become hot, set the switch of the fan controller higher, or move the fans to headers on the board.

## BIOS

Set these values after the flash, because a flash resets each setting to its default. The reason for each setting is in the [build checklist](../docs/build-checklist.md).

| Setting | Path | Value |
| --- | --- | --- |
| SATA Mode Selection | Advanced → SATA Configuration | AHCI |
| Aggressive LPM Support | Advanced → SATA Configuration | Disabled |
| S.M.A.R.T. Status Check | Advanced → SATA Configuration | Enabled |
| Hot Plug (all ports) | Advanced → SATA Configuration | Disabled. There is no hot-swap backplane |
| Restore AC Power Loss | Advanced → APM | Power On |
| Next Boot after AC Power Loss | Boot | Normal Boot |
| Wait For 'F1' If Error | Boot | Disabled |
| Boot Device Control | Boot → CSM | UEFI only. `Launch CSM` stays Enabled, but it gives no legacy paths |
| OS Type | Boot → Secure Boot | Other OS. Examine it again after each flash |
| EIST, CPU C-states | Advanced → CPU Configuration → CPU Power Management | Enabled (C1E, C3 and C6 enabled, Package C State Auto) |
| Boot Option 1 | Boot | The Intenso SSD. Examine it again after each flash |

## Installation

| Setting | Value | Reason |
| --- | --- | --- |
| Boot device | The Intenso SSD on `SATA6G_4` | Separate from the pool |
| Host name | `<NAS_HOSTNAME>` | |
| Time zone | `Europe/Luxembourg` | The default of the installer is US Pacific. Correct it before you schedule anything (*System Settings → General → Localization*) |

### The admin account

The default account `truenas_admin` was replaced and locked, and `root` is locked. To replace an admin account safely, add the new account first. Then remove the old one. Never rename the only account that you can log in with.

1. In *Credentials → Users → Add*, make the new user.
2. Put the user in the group `builtin_administrators`. That group gives the admin rights in the web UI, not the name of the user.
3. Log out.
4. Log in as the new user, and make sure that the user has admin access.
5. Only then, disable or delete the old account.

The properties of the account:

| Property | Value |
| --- | --- |
| Password | Set. The web UI uses it for the login, so "Disable Password" locks you out of the web UI |
| Shell | `zsh`. It must be a real shell: with `nologin`, the login to the web UI works and SSH fails |
| Home | `NAS_Storage/home/<user>`, on the pool. TrueNAS writes `authorized_keys` into the home |
| SSH key | The `id_ed25519` of the workstation, with a passphrase ([security.md](../docs/security.md)) |
| SMB access | Off |
| Sudo | *Allowed Sudo Commands (No Password)*: `ALL`. Nothing in *Allowed sudo commands* |

- Fill in only the sudo field without password. `ALL` in both fields makes two rules: `ALL=(ALL) ALL` and `ALL=(ALL) NOPASSWD: ALL`. In sudoers, the last match wins, so the password prompt can come back while the web UI looks correct. Test with `sudo -k; sudo true`. `sudo -l` shows the rules in their order.
- A change to the groups needs a new login. A change to the own sudo entry of the user does not.
- SSH accepts only keys. Set it at the service: in *System Settings → Services → SSH*, clear **Allow Password Authentication**. Prove that the key works first.
- The first login with `zsh` starts `zsh-newuser-install`. That proves that the home is real. Answer **(2)** for the `.zshrc` that Debian recommends. With (0) or (q), there is no permanent history. Increase `HISTSIZE` and `SAVEHIST` from 1000.
- Use one SSH key for each client device, not one for each server.
- The menu labels change between releases.

### API keys

- TrueNAS revokes an API key of a user the first time that the key arrives over plain HTTP. A reset gives a new string. TrueNAS judges the connection that it receives. So a client that uses `https://truenas.donny.lu` still loses its key if Caddy forwards over HTTP. That is why the upstream of Caddy for TrueNAS is HTTPS ([Caddy runbook](caddy.md), "Caddyfile").
- A key has the privileges of its user. So a key for a tool that only reads must be the key of a user with a read-only role, not of the admin account.
- The key of Homepage is the key of a dedicated user: *TrueNAS Access* → **Readonly Admin**, *Disable Password* on, no SMB, no shell, no SSH, and the home `/var/empty`. The API key is the only credential of that user. Readonly Admin can read all the configuration, not only the statistics of the widget. A narrower custom role is possible, but it is not set.

## Network

The NAS has static addresses for IPv4 and IPv6 since 31 August 2026.

| Stack | Value |
| --- | --- |
| IPv4 | `<NAS_IPV4>/24`. The reservation on the router stays, so that no other device gets the address |
| IPv6 | `<NAS_ULA>/64`, a static ULA. Its host part mirrors the IPv4 address |

- TrueNAS hides the fields for static addresses while DHCP is on. Clear DHCP to show them. From then on, both stacks are manual. *Autoconfigure IPv6* is not the control for this.
- Set the gateway and the nameservers in *Network → Global Configuration* at the same time. They came from DHCP. Without them, the NAS still answers on the LAN, but it cannot reach the outside. Remote access, certificate renewals and image pulls then fail.
- A change to an interface applies behind a timer that confirms or reverts. Do not confirm settings that you did not test. This machine has no IPMI, so a bad configuration that is confirmed needs a monitor and a keyboard.
- The internal DNS records point at the static ULA, not at a SLAAC address ([network.md](../docs/network.md)).
- The SLAAC addresses are stable privacy addresses, not the EUI-64 form. The EUI-64 form contains the MAC address, and each server that the NAS connects to over IPv6 sees the global address. Set `net.ipv6.conf.enp3s0.addr_gen_mode` to `3` in *System Settings → Advanced → Sysctl*. With `3`, the kernel picks a new random secret at each start, so the SLAAC addresses change at each reboot.
- The kernel makes the new addresses only when the interface starts again, so reboot after a change. A change with `sysctl -w` is lost at that reboot, before it applies. If the network card or its name changes, the key no longer matches, and the addresses go back to EUI-64 with no error. To check, `sysctl net.ipv6.conf.enp3s0.addr_gen_mode` must print `3`, and the output of `ip -6 addr show scope global` must have no `ff:fe`.

## Alerts through Telegram

The alerts deliver since 12 August 2026. E-mail was the first attempt. The alert service sets only the recipient. The outgoing mail is a separate setting (*System Settings → General → Email*). When it is not set, nothing is sent, with no error. The reason for Telegram is in [services.md](../docs/services.md).

1. In Telegram, send `/newbot` to **@BotFather**. It gives the bot token.
2. Send a message to your bot. Until you do, `getUpdates` returns an empty result.
3. Open `https://api.telegram.org/bot<TOKEN>/getUpdates`, and read `message.chat.id`. As an alternative, ask @userinfobot: for a private chat, your user ID is the chat ID.
4. In *Alert Settings*, add Telegram with the token and the chat ID.
5. Test at the level `INFO`.
6. Set the level to `Warning`.

`getMe` confirms which bot a token is for. If the updates stay empty, use `getWebhookInfo`.

To isolate a failure:

```bash
curl -s "https://api.telegram.org/bot<TOKEN>/getMe"
curl -s "https://api.telegram.org/bot<TOKEN>/sendMessage?chat_id=<ID>&text=hello"
sudo journalctl --since "10 min ago" | grep -i telegram
```

The token is a credential. Do not put it in this repository. It is also in the config backup. The alert messages send drive serials and host names off the network ([services.md](../docs/services.md)).

## Config backup

The config backup is made by hand: *System Settings → General → Manage Configuration → Download File*. It is kept on the workstation. A scheduled export is planned.

- The file holds password hashes and the bot token. Never put it in this repository.
- It contains the database of the middleware: users, groups, datasets, shares, services and apps.
- It does not contain the home directories or other data on the pool.

## The UPS service

This host is the NUT master ([power.md](../docs/power.md)). The UPS service stays stopped until the USB cable of the UPS is connected to this host.
