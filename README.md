# phev2mqtt on Proxmox Debian 13

A complete guide for migrating [buxtronix/phev2mqtt](https://github.com/buxtronix/phev2mqtt) from a Raspberry Pi to a Debian 13 VM running on Proxmox.

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
- Updating phev2mqtt when new versions are released
- Useful diagnostic commands
- Known issues and fixes

## Hardware used

- Proxmox server
- TP-Link Archer T3U Plus (AC1300) USB WiFi adapter
- Mitsubishi Outlander PHEV
- Home Assistant with Mosquitto MQTT broker

## Files

| File | Description |
|---|---|
| [phev2mqtt-deb13-guide.md](./phev2mqtt-deb13-guide.md) | Complete setup guide |
| [lovelace-card.yaml](./lovelace-card.yaml) | Home Assistant Lovelace card for the PHEV Gateway dashboard |

## Credits

- [buxtronix/phev2mqtt](https://github.com/buxtronix/phev2mqtt) — the original project this guide is based on
- [RinCat/RTL88x2BU-Linux-Driver](https://github.com/RinCat/RTL88x2BU-Linux-Driver) — WiFi driver used in this guide
- [Claude by Anthropic](https://claude.ai) — assisted in writing and structuring this guide

## Disclaimer

This guide is provided "as is" without warranty of any kind. Follow it at your own risk. The author is not responsible for any damage to your vehicle, home automation system, or any other hardware or software. Always make sure you have backups and Proxmox snapshots before making changes.

This guide is not affiliated with or endorsed by Mitsubishi Motors, Proxmox, Home Assistant, or any other mentioned product or service.

## Licence

MIT
