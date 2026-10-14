# phev2mqtt on Proxmox Debian 13

A complete guide for migrating [buxtronix/phev2mqtt](https://github.com/buxtronix/phev2mqtt) from a Raspberry Pi to a Debian 13 VM running on Proxmox. It builds [corautem/phev2mqtt](https://github.com/corautem/phev2mqtt), a fork with connection fixes and optional extra sensors.

## What this guide covers

- Creating a Debian 13 VM in Proxmox
- USB WiFi adapter passthrough (TP-Link Archer T3U Plus / RTL88x2BU)
- Installing the RTL88x2BU driver
- Connecting to the Mitsubishi Outlander PHEV WiFi hotspot
- Building and installing phev2mqtt from source
- Configuring phev2mqtt as a systemd service
- Migrating from Raspberry Pi (stopping the old instance)
- Home Assistant MQTT integration and auto-discovery
- WiFi status sensors, reconnect button and WiFi enable/disable switch in Home Assistant
- Updating phev2mqtt with every build kept, and rolling back in one step (script or Home Assistant button)
- Optional extra car sensors: power switch, security alarm, charge type, the car's own charge and climate timers
- Useful diagnostic commands
- Known issues and fixes

## Hardware used

- Proxmox server
- TP-Link Archer T3U Plus (AC1300) USB WiFi adapter
- Mitsubishi Outlander PHEV
- Home Assistant with Mosquitto MQTT broker

## Screenshots

![PHEV Dashboard](./screenshots/dashboard-mobile.png)
![PHEV Gateway Card](./screenshots/gateway-card.png)

## Files

| File | Description |
|---|---|
| [phev2mqtt_guide.md](./phev2mqtt_guide.md) | Complete setup guide |
| [lovelace-card.yaml](./lovelace-card.yaml) | Home Assistant Lovelace card for the PHEV Gateway dashboard |
| [configuration.yaml](./configuration.yaml) | Home Assistant MQTT sensors, switch and buttons for the gateway |
| [templates.yaml](./templates.yaml) | Home Assistant template sensor for a readable service uptime |

## Credits

- [buxtronix/phev2mqtt](https://github.com/buxtronix/phev2mqtt) — the original project this guide is based on
- [Anwo91/phev-remote-plus](https://github.com/Anwo91/phev-remote-plus) — register decoding behind the optional extra sensors
- [RinCat/RTL88x2BU-Linux-Driver](https://github.com/RinCat/RTL88x2BU-Linux-Driver) — WiFi driver used in this guide
- [Claude by Anthropic](https://claude.ai) — assisted in writing and structuring this guide

## Disclaimer

This guide is provided "as is" without warranty of any kind. Follow it at your own risk. The author is not responsible for any damage to your vehicle, home automation system, or any other hardware or software. Always make sure you have backups and Proxmox snapshots before making changes.

This guide is not affiliated with or endorsed by Mitsubishi Motors, Proxmox, Home Assistant, or any other mentioned product or service.

## Licence

MIT
