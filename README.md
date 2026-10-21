# openwrt-new-device-alert

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![OpenWrt](https://img.shields.io/badge/OpenWrt-24.10-00B5E2?logo=openwrt&logoColor=white)](https://openwrt.org/)
[![Shell](https://img.shields.io/badge/shell-POSIX%20sh-4EAA25?logo=gnubash&logoColor=white)](src/lanwatch-leases)
[![Telegram](https://img.shields.io/badge/notifications-Telegram-26A5E4?logo=telegram&logoColor=white)](https://core.telegram.org/bots)
[![Dependencies](https://img.shields.io/badge/dependencies-curl-blue)](#requirements)

🇬🇧 English | 🇷🇺 [Русский](README.ru.md)

A tiny Telegram notifier for **new devices on your OpenWrt router**, covering the main **and the guest** network. One POSIX shell script: no scanning, no database, no daemon, no web UI.

## Features

- 🆕 Alert when a new MAC address gets a DHCP lease
- 🔄 Alert when a known device changes its IP (can be turned off)
- 🔙 Alert when a device returns after being absent for N days (threshold configurable, can be turned off)
- Separate labels for the main and the guest network
- Several Telegram recipients, one bot can serve several routers (router label at the end of each message)
- Retry queue: if Telegram or the internet is down, messages are stored and delivered later
- Optional device vendor lookup, marker for random (private) MAC addresses
- Very low load: a ~1 ms file read once a minute; flash is written only when something changes

## How it works

Once a minute cron runs `lanwatch-leases`. It reads `/tmp/dhcp.leases` (the lease file that dnsmasq maintains anyway), compares it with a small local database in `/etc/lanwatch/devices.db` and sends a Telegram message when something noteworthy happened. The network is never scanned.

> Why cron and not the `dhcpscript` hook of dnsmasq? On OpenWrt 24.10 dnsmasq runs in a sandbox and the hook cannot read its own config file. Reading the lease file from cron avoids the problem and does not touch the dnsmasq configuration.

Example message:

```
🆕 New device on the network

Name: iPhone
IP: 192.168.50.23
MAC: 3c:a0:67:57:0d:bb
Vendor: Samsung Electro-Mechanics
Network: Guest network

🛜 Router 1
```

(Message texts are in Russian by default; edit the strings in `src/lanwatch-leases` to translate them.)

## Requirements

- OpenWrt 24.10 (tested). OpenWrt 25.x (package manager `apk`) should work, but has not been tested
- dnsmasq as the DHCP server (default)
- Packages `curl` and `ca-bundle`
- A Telegram bot token and your chat id(s)

## Install

The full step-by-step manual (finding interfaces, creating the bot, testing, upgrading, troubleshooting) is in [docs/INSTALL.md](docs/INSTALL.md).

## Configuration

All settings live in `/etc/lanwatch/config` (see [src/config.example](src/config.example)).

| Option | Meaning |
|---|---|
| `BOT_TOKEN` | Bot token from @BotFather |
| `CHAT_IDS` | Recipients separated by spaces (group ids are usually negative) |
| `ROUTER_NAME` | Label added to the end of each message (empty = none) |
| `MAIN_IF`, `GUEST_IF` | Interfaces that serve DHCP, e.g. `br-lan`, `br-guest` |
| `VENDOR_LOOKUP` | `1` = look up the vendor via api.macvendors.com (only the first 3 MAC bytes are sent), `0` = off |
| `EVENT_IP` | `1` = alert on IP change, `0` = off |
| `RETURN_DAYS` | Alert when a device returns after more than N days, `0` = off |

## Pros and cons compared to full network scanners

Full scanners solve a bigger task. This project solves a narrow one: *tell me when something new shows up*.

**Pros**

- Runs on the router itself; no extra server, Docker or web service to maintain
- Sees the guest network, because it reads the router's own DHCP leases (no VLAN or firewall tricks for a scanner)
- Always shows the current IP of a device: changes are tracked by MAC
- No active scanning: no extra traffic, no port probes
- Tiny: one shell script and `curl`; negligible CPU and RAM
- Notifications survive outages (retry queue), several recipients, several routers with one bot

**Cons**

- Only devices that use DHCP are seen; a device with a static IP is invisible
- No web interface, charts, history or "online / offline" status
- No port scanning, OS or service fingerprinting
- Up to one minute of delay (cron interval)
- Phones with private (random) MAC addresses can look like new devices
- Depends on the dnsmasq lease file; other DHCP servers are not supported
- Device names are reduced to Latin letters, digits and `. _ ( ) -`

## Repository layout

```
.
├── README.md            English overview
├── README.ru.md         Russian overview
├── LICENSE              MIT
├── src/
│   ├── lanwatch-leases  the script (goes to /usr/bin/ on the router)
│   └── config.example   example config (goes to /etc/lanwatch/config)
└── docs/
    ├── INSTALL.md       detailed manual (English)
    └── INSTALL.ru.md    detailed manual (Russian)
```

## Security notes

- The bot token is stored in plain text in `/etc/lanwatch/config` (mode `600`, readable by root only). Never commit it
- The token is part of the request URL, so it can briefly be visible in the process list on the router
- With `VENDOR_LOOKUP='1'` the first three bytes of the MAC address of *new* devices are sent to api.macvendors.com

## License

[MIT](LICENSE). The software is provided "as is", without warranty of any kind.
