# phev2mqtt on Proxmox Debian 13 — Complete Setup Guide

> **Disclaimer:** This guide is provided "as is" without warranty of any kind. Follow it at your own risk. The author is not responsible for any damage to your vehicle, home automation system, or any other hardware or software. Always make sure you have backups and Proxmox snapshots before making changes. This guide is not affiliated with or endorsed by Mitsubishi Motors, Proxmox, Home Assistant, or any other mentioned product or service.

This guide documents the full process of migrating the [buxtronix/phev2mqtt](https://github.com/buxtronix/phev2mqtt) service from a Raspberry Pi to a Debian 13 VM running on Proxmox, including WiFi adapter setup, Home Assistant integration, and automation scripts.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Create the Debian 13 VM in Proxmox](#2-create-the-debian-13-vm-in-proxmox)
3. [First Steps After Fresh Install](#3-first-steps-after-fresh-install)
4. [USB WiFi Adapter Passthrough](#4-usb-wifi-adapter-passthrough)
5. [Install RTL88x2BU WiFi Driver](#5-install-rtl88x2bu-wifi-driver)
6. [Connect to the PHEV WiFi Hotspot](#6-connect-to-the-phev-wifi-hotspot)
7. [Fix Routing — Keep Internet on Ethernet](#7-fix-routing--keep-internet-on-ethernet)
8. [Build and Install phev2mqtt](#8-build-and-install-phev2mqtt)
9. [Test Connection to the Car](#9-test-connection-to-the-car)
10. [Configure phev2mqtt as a Systemd Service](#10-configure-phev2mqtt-as-a-systemd-service)
11. [Disable phev2mqtt on the Raspberry Pi](#11-disable-phev2mqtt-on-the-raspberry-pi)
12. [Make wpa_supplicant Start on Boot](#12-make-wpa_supplicant-start-on-boot)
13. [Home Assistant Integration](#13-home-assistant-integration)
14. [WiFi Status Scripts and MQTT Sensors](#14-wifi-status-scripts-and-mqtt-sensors)
15. [Home Assistant Lovelace Card](#15-home-assistant-lovelace-card)
16. [How to Update and Roll Back phev2mqtt](#16-how-to-update-and-roll-back-phev2mqtt)
17. [Useful Diagnostic Commands](#17-useful-diagnostic-commands)
18. [Known Issues and Fixes](#18-known-issues-and-fixes)
19. [Self-Healing Watchdog](#19-self-healing-watchdog)
20. [Optional Extra Sensors](#20-optional-extra-sensors)
21. [Vehicle Settings](#21-vehicle-settings)

---

## 1. Prerequisites

Before starting, make sure you have:

- A Proxmox server up and running
- A Debian 13 (Trixie) ISO downloaded
- A **TP-Link Archer T3U Plus (AC1300)** USB WiFi adapter (or similar RTL88x2BU chipset)
- A Mitsubishi Outlander PHEV with WiFi hotspot credentials (SSID and password)
- Home Assistant running with the **Mosquitto MQTT broker** add-on installed
- MQTT broker credentials (username and password)
- Your Raspberry Pi running phev2mqtt (to be decommissioned)

---

## 2. Create the Debian 13 VM in Proxmox

1. Upload the Debian 13 ISO to Proxmox local storage
2. Create a new VM with the following recommended settings:
   - **OS**: Linux, Debian
   - **CPU**: 1–2 vCPUs (real-world usage settles around 4%)
   - **RAM**: 1–2GB recommended (real-world usage settles around 700MB with Debian 13 base system + all services running — 512MB is too tight)
   - **Disk**: 8GB is sufficient
   - **Network**: VirtIO, bridged to your LAN
3. Enable **QEMU Guest Agent** in the VM options (VM → Options → QEMU Guest Agent → Enable)
4. Install Debian 13 following the standard installer
5. Set hostname to something memorable e.g. `phev2mqtt-deb13`

---

## 3. First Steps After Fresh Install

After Debian is installed and you can log in as root, run the following:

### Update the system

```bash
apt update && apt upgrade -y
```

This fetches the latest package lists and upgrades all installed packages.

### Install essential tools

```bash
apt install -y curl wget git vim nano htop net-tools sudo ufw unzip rsync
```

Key tools explained:

- `curl` / `wget` — download files from the internet
- `git` — needed to clone the phev2mqtt repository
- `nano` / `vim` — text editors for editing config files
- `htop` — interactive process monitor
- `ufw` — simple firewall manager

### Set timezone

```bash
timedatectl set-timezone Europe/London
```

Replace `Europe/London` with your timezone. List all available timezones with:

```bash
timedatectl list-timezones
```

### Install QEMU Guest Agent

This allows Proxmox to properly communicate with the VM (shows IP, allows clean shutdown, snapshot quiescing):

```bash
apt install -y qemu-guest-agent
systemctl start qemu-guest-agent
```

> **Note:** You may see a warning about "no installation config" — this is harmless. The agent is statically enabled via udev and will start automatically.

### Configure firewall

```bash
ufw allow OpenSSH
ufw enable
```

### Take a Proxmox snapshot

Before proceeding, take a snapshot of the VM from the Proxmox UI as a clean restore point.

---

## 4. USB WiFi Adapter Passthrough

The TP-Link Archer T3U Plus needs to be passed through from the Proxmox host to the VM so the VM has direct access to the hardware.

### Choosing a WiFi adapter — native vs out-of-tree drivers

The adapter used in this guide (TP-Link Archer T3U Plus / RTL88x2BU) requires an **out-of-tree driver** that needs to be manually installed. If you are buying a new adapter, consider choosing one with a **native in-kernel driver** instead — these work plug and play with no extra installation steps.

The following chipsets have native Linux kernel support and are recommended if you are purchasing a new adapter:

| Chipset               | Standard         | In-kernel since    | Notes                                  |
| --------------------- | ---------------- | ------------------ | -------------------------------------- |
| **Mediatek mt7921au** | WiFi 6 / AXE3000 | Kernel 5.18 (2022) | Best overall choice, very stable       |
| **Mediatek mt7612u**  | WiFi 5 / AC1200  | Kernel 4.19 (2018) | Reliable, widely available             |
| **Mediatek mt7610u**  | WiFi 5 / AC600   | Kernel 4.19 (2018) | Budget option, USB 2.0                 |
| **Realtek rtl8812bu** | WiFi 5 / AC1200  | Kernel 6.2 (2023)  | Kernel 6.12+ recommended               |
| **Realtek rtl8812au** | WiFi 5 / AC1200  | Kernel 6.14 (2025) | Previously required out-of-tree driver |
| **Mediatek mt7601u**  | WiFi 4 / N150    | Kernel 4.2 (2015)  | Basic, managed mode only               |

> **Recommended pick:** The **Mediatek mt7921au** chipset (e.g. Panda PAU0F, EDUP EP-AX1672) is the best choice for a new purchase — WiFi 6, very stable, plug and play on Debian 13.

> For a comprehensive and up-to-date list of compatible adapters and models, see the [morrownr/USB-WiFi](https://github.com/morrownr/USB-WiFi) repository which is the definitive reference for Linux USB WiFi adapter compatibility.

> **Note on the RTL88x2BU (this guide's adapter):** The RTL88x2BU chipset used in the TP-Link Archer T3U Plus does have in-kernel support since kernel 6.2, but kernel 6.12+ is recommended for stability. Debian 13 ships with kernel 6.1 by default, which is why the out-of-tree driver from RinCat is used in this guide. If you are running a newer kernel (6.12+) you may find the adapter works without the manual driver installation — check with `dmesg | grep rtw88` after plugging it in.

### Passthrough the adapter in Proxmox UI:

1. Select your VM → **Hardware** → **Add** → **USB Device**
2. Choose **Use USB Vendor/Device ID**
3. Select the TP-Link adapter from the list (Vendor ID: `2357`, Device ID: `0138`)
4. Click **Add** and start/restart the VM

### Verify the adapter is visible inside the VM:

```bash
apt install -y usbutils
lsusb
```

You should see:

```
Bus 010 Device 002: ID 2357:0138 TP-Link 802.11ac NIC
```

---

## 5. Install RTL88x2BU WiFi Driver

The TP-Link Archer T3U Plus uses the **RTL88x2BU** chipset which is not included in the Linux kernel by default on Debian 13. You need to install a third-party driver.

### Install build dependencies

```bash
apt install -y git dkms build-essential linux-headers-$(uname -r)
```

- `dkms` — automatically rebuilds the driver after kernel updates
- `build-essential` — C compiler and build tools
- `linux-headers` — kernel headers needed to compile the driver

### Clone and install the driver

```bash
git clone "https://github.com/RinCat/RTL88x2BU-Linux-Driver.git" /usr/src/rtl88x2bu-git
sed -i 's/PACKAGE_VERSION="@PKGVER@"/PACKAGE_VERSION="git"/g' /usr/src/rtl88x2bu-git/dkms.conf
dkms add -m rtl88x2bu -v git
dkms autoinstall
```

### Reboot

```bash
reboot
```

### Verify the adapter is detected

```bash
ip link show
```

You should see a new interface named `wlx` followed by the MAC address, e.g. `wlx6c4cbc0a41b0`. This is your WiFi interface name — **note it down**, you will need it throughout this guide. Replace `YOUR_WIFI_INTERFACE_NAME` in all commands and config files below with this value.

### Disable WiFi power saving (prevents dropouts)

```bash
iw dev YOUR_WIFI_INTERFACE_NAME set power_save off
```

To make this permanent, create a systemd service. The service must wait until the WiFi interface is fully enumerated by the kernel before running — using `After=network.target` is not sufficient and will cause the service to fail with `No such device`:

```bash
nano /etc/systemd/system/wifi-powersave-off.service
```

```ini
[Unit]
Description=Disable WiFi power save
After=sys-subsystem-net-devices-YOUR_WIFI_INTERFACE_NAME.device
Requires=sys-subsystem-net-devices-YOUR_WIFI_INTERFACE_NAME.device

[Service]
Type=oneshot
ExecStart=/sbin/iw dev YOUR_WIFI_INTERFACE_NAME set power_save off
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
systemctl enable wifi-powersave-off
systemctl start wifi-powersave-off
```

Verify it applied:

```bash
iw dev YOUR_WIFI_INTERFACE_NAME get power_save
# Expected output: Power save: off
```

> **Why `Requires=sys-subsystem-net-devices-...`?** Systemd generates a device unit for every network interface from udev events. By binding the service to this unit, systemd guarantees the service only runs once the kernel has registered the interface — eliminating the race condition that causes `No such device` errors at boot.

> **Driver log messages explained:** After installing the driver you will see messages like `Dump efuse in suspend`, `Invalid rate 0x0`, `power state unchange` in dmesg. These are known quirks of this driver and are harmless. The important line confirming success is:
> `rtw_ndev_init(wlan0) mac_addr=xx:xx:xx:xx:xx:xx`

### Disable USB autosuspend (a second, separate power-save layer)

The `power_save off` setting above only disables the adapter's **802.11 Link Power Save** — a WiFi-protocol-level sleep mode. Linux's **USB autosuspend** is a completely separate mechanism that can independently suspend the USB WiFi dongle itself after a period of inactivity, regardless of the 802.11 setting. Leaving it enabled is a second possible cause of the radio going unresponsive (see [Known Issues](#18-known-issues-and-fixes)).

Check the current setting:

```bash
cat /sys/module/usbcore/parameters/autosuspend
# A value other than -1 (commonly 2) means autosuspend is enabled
```

Disable it permanently via a kernel boot parameter:

```bash
nano /etc/default/grub
```

Add `usbcore.autosuspend=-1` to the `GRUB_CMDLINE_LINUX_DEFAULT` line, e.g.:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet usbcore.autosuspend=-1"
```

```bash
update-grub
reboot
```

Verify after reboot:

```bash
cat /sys/module/usbcore/parameters/autosuspend
# Expected: -1
```

> **Why disable it globally instead of per-device?** A per-device fix requires a udev rule matching the dongle's exact vendor/product ID. On a single-purpose headless VM with no battery to save power for, disabling USB autosuspend system-wide is simpler and has no meaningful downside.

---

## 6. Connect to the PHEV WiFi Hotspot

The PHEV creates its own WiFi hotspot. You need to connect the VM's WiFi adapter to it using `wpa_supplicant`.

### Install wpa_supplicant

```bash
apt install -y wpasupplicant
```

### Create the WiFi config file

```bash
nano /etc/wpa_supplicant/wpa_supplicant.conf
```

```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
    ssid="YOUR_PHEV_SSID"
    scan_ssid=1
    psk="YOUR_PHEV_PASSWORD"
}
```

Replace `YOUR_PHEV_SSID` and `YOUR_PHEV_PASSWORD` with your car's WiFi credentials (found in the Mitsubishi documentation or app).

### Start wpa_supplicant manually (for testing)

```bash
wpa_supplicant -B -i YOUR_WIFI_INTERFACE_NAME -c /etc/wpa_supplicant/wpa_supplicant.conf
```

- `-B` runs it in the background
- `-i` specifies the interface
- `-c` specifies the config file

### Configure systemd-networkd to assign an IP via DHCP on the WiFi interface

```bash
nano /etc/systemd/network/20-wifi.network
```

```ini
[Match]
Name=YOUR_WIFI_INTERFACE_NAME

[Network]
DHCP=yes

[DHCP]
RouteMetric=200
```

> **Important:** `RouteMetric=200` ensures the PHEV WiFi route has lower priority than your ethernet (metric 100), so internet traffic always goes through ethernet and not the car's hotspot.

```bash
systemctl restart systemd-networkd
```

### Verify the IP address

```bash
ip addr show YOUR_WIFI_INTERFACE_NAME
```

You should see `inet 192.168.8.47` — this is the IP the PHEV always assigns to connected clients.

---

## 7. Fix Routing — Keep Internet on Ethernet

When the WiFi adapter connects to the PHEV, it adds a default route via `192.168.8.46` (the car's IP) which breaks internet connectivity. The `RouteMetric=200` setting above prevents this permanently, but if you ever lose internet while connected to the car, run:

```bash
# Check current routes
ip route show

# Remove the unwanted PHEV default route if present
ip route del default via 192.168.8.46 dev YOUR_WIFI_INTERFACE_NAME

# Verify internet is restored
ping 1.1.1.1
```

---

## 8. Build and Install phev2mqtt

phev2mqtt is written in Go and needs to be compiled from source.

### Install Go

> **Do not use the Debian package** — it is often outdated. Download directly from the Go website:

```bash
wget https://go.dev/dl/go1.23.5.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.23.5.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> /etc/profile
source /etc/profile
go version
```

### Clone and build phev2mqtt

This guide uses [corautem/phev2mqtt](https://github.com/corautem/phev2mqtt), a fork of [buxtronix/phev2mqtt](https://github.com/buxtronix/phev2mqtt) with connection fixes and optional extra sensors (see [Section 20](#20-optional-extra-sensors)). With no options switched on it behaves like the original.

```bash
git clone https://github.com/corautem/phev2mqtt.git
cd phev2mqtt
go build
```

This compiles the binary. It may take a minute or two.

### Install the binary

```bash
cp phev2mqtt /usr/local/bin/
chmod +x /usr/local/bin/phev2mqtt
phev2mqtt -h
```

> **Later updates** go through the install script in [Section 16](#16-how-to-update-and-roll-back-phev2mqtt), which keeps every build so you can switch back to an earlier one in seconds.

---

## 9. Test Connection to the Car

Before setting up the service, test that phev2mqtt can connect to the car and receive data.

### Register the VM as a client to the car

> **Important:** The VM must be registered with the car before it can receive any data. The device can connect to the car's WiFi but will not send or receive data until registration is complete. Registration can also fail if the WiFi signal is weak — make sure the car is close enough to get a strong signal before attempting this step.

1. Verify your WiFi connection to the car is established and you have the correct IP:

```bash
ip addr show YOUR_WIFI_INTERFACE_NAME
```

You should see `inet 192.168.8.47`.

2. Follow the [Mitsubishi instructions](https://www.mitsubishi-motors.com/en/products/outlander_phev/app/remote/) to put the car into registration mode ("Setup Your Vehicle").

3. Run the registration command:

```bash
/usr/local/bin/phev2mqtt client register
```

You should shortly see a message indicating successful registration. If it fails, move the car closer, ensure the signal is strong, and try again.

> **Note:** Registration only needs to be done once. After successful registration the VM will be remembered by the car and will not need to be registered again — unless you delete registered devices from the car via the Mitsubishi app or car settings, in which case the registration process will need to be repeated from scratch.

### Watch live data from the car

```bash
cd ~/phev2mqtt
./phev2mqtt client watch
```

You should see `%PHEV_TCP_CONNECTED%` followed by a stream of register updates showing battery level, door status, AC status etc.

Press `Ctrl+C` to stop.

### Test MQTT connection

```bash
apt install -y mosquitto-clients

mosquitto_pub -h HA_IP_ADDRESS -p 1883 \
  -u YOUR_MQTT_USER -P YOUR_MQTT_PASS \
  -t test/topic -m "hello"
```

If no error is returned, your MQTT credentials are correct.

---

## 10. Configure phev2mqtt as a Systemd Service

### Create the service file

```bash
nano /etc/systemd/system/phev2mqtt.service
```

```ini
[Unit]
Description=phev2mqtt service
After=network.target
StartLimitIntervalSec=5

[Service]
Type=exec
ExecStart=/usr/local/bin/phev2mqtt --config=/dev/null client mqtt --mqtt_server tcp://HA_IP_ADDRESS:1883 --mqtt_username YOUR_MQTT_USER --mqtt_password YOUR_MQTT_PASS
Restart=always
RestartSec=30s
SyslogIdentifier=phev2mqtt
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

> **Important — Raspberry Pi conflict:** If your Raspberry Pi is still running phev2mqtt and connected to the same MQTT broker, both instances will use the same client ID (`phev2mqtt`) and keep kicking each other off. You will see `write: broken pipe` errors in the logs and `Client phev2mqtt already connected, closing old connection` in the Mosquitto broker logs. **Stop and disable phev2mqtt on the Pi before starting the service on the VM** — see Section 11.

### Enable and start the service

```bash
systemctl daemon-reload
systemctl enable phev2mqtt
systemctl start phev2mqtt
```

### Verify it is running

```bash
systemctl status phev2mqtt
journalctl -f -u phev2mqtt -o cat
```

You should see `%PHEV_TCP_CONNECTED%` in the logs.

---

## 11. Disable phev2mqtt on the Raspberry Pi

The PHEV only allows one WiFi client connection at a time. If the Pi is still running phev2mqtt it will conflict with the VM — they will keep kicking each other off (you will see `Client phev2mqtt already connected, closing old connection` in the MQTT broker logs).

SSH into your Raspberry Pi and run:

```bash
sudo systemctl stop phev2mqtt
sudo systemctl disable phev2mqtt
```

This stops the service immediately and prevents it from starting on reboot.

---

## 12. Make wpa_supplicant Start on Boot

The wpa_supplicant instance started manually in step 6 will not survive a reboot. Create a dedicated systemd service for it:

```bash
nano /etc/systemd/system/wpa_supplicant-phev.service
```

```ini
[Unit]
Description=WPA Supplicant for PHEV WiFi
After=sys-subsystem-net-devices-YOUR_WIFI_INTERFACE_NAME.device
BindsTo=sys-subsystem-net-devices-YOUR_WIFI_INTERFACE_NAME.device

[Service]
Type=forking
ExecStart=/usr/sbin/wpa_supplicant -B -i YOUR_WIFI_INTERFACE_NAME -c /etc/wpa_supplicant/wpa_supplicant.conf
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

> **Important:** Before enabling this service, kill any manually started wpa_supplicant instances for this interface:

```bash
# Find the PID of the manual instance
ps aux | grep wpa_supplicant

# Kill it (replace PID with the actual number)
kill PID
```

```bash
systemctl daemon-reload
systemctl enable wpa_supplicant-phev
systemctl start wpa_supplicant-phev
systemctl status wpa_supplicant-phev
```

### Full boot sequence after this step:

1. VM boots
2. `wpa_supplicant-phev` starts → connects WiFi adapter to PHEV hotspot
3. `systemd-networkd` assigns IP `192.168.8.47` to the WiFi interface
4. `phev2mqtt` starts and connects to the car and MQTT broker (retries every 30 seconds if it fails)
5. Home Assistant receives car data automatically

---

## 13. Home Assistant Integration

### MQTT Discovery

phev2mqtt supports Home Assistant MQTT Discovery by default. After the service starts successfully, entities will appear automatically in HA. Search for "phev" in:

- **Settings → Devices & Services → MQTT**
- **Settings → Entities**

---

## 14. WiFi Status Scripts and MQTT Sensors

> **This section is optional but recommended.** It sets up scripts that publish VM WiFi stats and phev2mqtt service state to MQTT every 30 seconds and also immediately on any state change, and adds the ability to reconnect, enable/disable WiFi, or restart the phev2mqtt process directly from Home Assistant.

### Status script — publishes WiFi and service stats to MQTT

```bash
nano /usr/local/bin/phev-wifi-status.sh
```

```bash
#!/bin/bash

MQTT_HOST="HA_IP_ADDRESS"
MQTT_USER="YOUR_MQTT_USER"
MQTT_PASS="YOUR_MQTT_PASS"
IFACE="YOUR_WIFI_INTERFACE_NAME"

# WiFi stats
IP=$(ip addr show $IFACE | grep 'inet ' | awk '{print $2}' | cut -d/ -f1)
SIGNAL=$(iw dev $IFACE link | grep 'signal' | awk '{print $2}')
SSID=$(iw dev $IFACE link | grep 'SSID' | awk '{print $2}')
STATUS=$(cat /sys/class/net/$IFACE/operstate)
ENABLED=$([ "$(cat /sys/class/net/$IFACE/operstate 2>/dev/null)" != "down" ] && systemctl is-active --quiet wpa_supplicant-phev && echo "ON" || echo "OFF")

# phev2mqtt service stats
SVC_STATE=$(systemctl is-active phev2mqtt)
SVC_UPTIME=$(systemctl show phev2mqtt --property=ActiveEnterTimestamp \
  | cut -d= -f2 | xargs -I{} date -d "{}" +%s 2>/dev/null)
NOW=$(date +%s)
if [ -n "$SVC_UPTIME" ] && [ "$SVC_UPTIME" -gt 0 ] 2>/dev/null; then
  SVC_UPTIME_SECS=$((NOW - SVC_UPTIME))
else
  SVC_UPTIME_SECS=0
fi
# Which build /usr/local/bin/phev2mqtt points to (see Section 16)
SVC_BUILD=$(basename "$(readlink -f /usr/local/bin/phev2mqtt)")

# Publish WiFi topics
mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/wifi/ip"      -m "${IP:-unavailable}"
mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/wifi/signal"  -m "${SIGNAL:-unavailable}"
mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/wifi/ssid"    -m "${SSID:-unavailable}"
mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/wifi/status"  -m "${STATUS:-unavailable}"
mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/wifi/enabled" -m "$ENABLED"

# Publish service topics
mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/service/state"  -m "$SVC_STATE"
mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/service/uptime" -m "$SVC_UPTIME_SECS"
mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/service/build"  -m "$SVC_BUILD"
```

```bash
chmod +x /usr/local/bin/phev-wifi-status.sh
```

### Reconnect script — restarts WiFi and phev2mqtt

```bash
nano /usr/local/bin/phev-reconnect.sh
```

```bash
#!/bin/bash

MQTT_HOST="HA_IP_ADDRESS"
MQTT_USER="YOUR_MQTT_USER"
MQTT_PASS="YOUR_MQTT_PASS"

mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/wifi/status" -m "reconnecting"

systemctl restart wpa_supplicant-phev
sleep 10
systemctl restart phev2mqtt

# Trigger immediate status update after reconnect
/usr/local/bin/phev-wifi-status.sh
```

```bash
chmod +x /usr/local/bin/phev-reconnect.sh
```

### WiFi enable/disable script

This allows you to turn off the VM's WiFi so another device (phone, laptop) can connect to the car, then turn it back on to reconnect the VM.

```bash
nano /usr/local/bin/phev-wifi-control.sh
```

```bash
#!/bin/bash

MQTT_HOST="HA_IP_ADDRESS"
MQTT_USER="YOUR_MQTT_USER"
MQTT_PASS="YOUR_MQTT_PASS"
IFACE="YOUR_WIFI_INTERFACE_NAME"
COMMAND=$1

case "$COMMAND" in
  ON)
    mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/wifi/enabled" -m "OFF"
    ip link set $IFACE up
    systemctl start wpa_supplicant-phev
    sleep 15
    systemctl start phev2mqtt
    mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/wifi/enabled" -m "ON"
    /usr/local/bin/phev-wifi-status.sh
    ;;
  OFF)
    mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/wifi/enabled" -m "OFF"
    systemctl stop phev2mqtt
    systemctl stop wpa_supplicant-phev
    ip link set $IFACE down
    /usr/local/bin/phev-wifi-status.sh
    ;;
esac
```

```bash
chmod +x /usr/local/bin/phev-wifi-control.sh
```

### WiFi driver reload script

Sometimes the USB WiFi adapter's firmware gets stuck in **Link Power Save (LPS)** mode (see [Known Issues](#18-known-issues-and-fixes)) badly enough that even restarting `wpa_supplicant` and phev2mqtt doesn't recover it — the radio itself is wedged. Reloading the kernel driver module resets the radio firmware without rebooting the whole VM, and is much faster (~20s) than a full reboot. This is not a "reset" in the router sense — no WiFi settings are erased or changed, it just briefly drops and automatically re-establishes the same connection using the existing saved credentials.

```bash
nano /usr/local/bin/phev-hardreset.sh
```

```bash
#!/bin/bash

MQTT_HOST="HA_IP_ADDRESS"
MQTT_USER="YOUR_MQTT_USER"
MQTT_PASS="YOUR_MQTT_PASS"
IFACE="YOUR_WIFI_INTERFACE_NAME"

mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/wifi/status" -m "resetting"

systemctl stop phev2mqtt
systemctl stop wpa_supplicant-phev

# Detect whichever WiFi driver module is currently loaded (the out-of-tree
# 88x2bu DKMS driver on kernel 6.1, or the in-kernel rtw88_8822bu on 6.12+).
MODULE=$(lsmod | awk '/^88x2bu|^rtw88_8822bu/ {print $1; exit}')

if [ -n "$MODULE" ]; then
  ip link set $IFACE down 2>/dev/null
  modprobe -r "$MODULE"
  sleep 2
  modprobe "$MODULE"
  sleep 5
else
  echo "WiFi driver module not found, skipping reload"
fi

iw dev $IFACE set power_save off
systemctl start wpa_supplicant-phev
sleep 15
systemctl start phev2mqtt

/usr/local/bin/phev-wifi-status.sh
```

```bash
chmod +x /usr/local/bin/phev-hardreset.sh
```

> **If this doesn't fix it either:** the module reload resets the driver's software state but doesn't power-cycle the USB device itself. If the radio is still stuck afterwards, a full VM reboot (below) forces the kernel to fully re-enumerate the USB device, which a plain module reload sometimes can't do.

### MQTT listener script — receives commands from Home Assistant

```bash
nano /usr/local/bin/phev-mqtt-listener.sh
```

```bash
#!/bin/bash

MQTT_HOST="HA_IP_ADDRESS"
MQTT_USER="YOUR_MQTT_USER"
MQTT_PASS="YOUR_MQTT_PASS"

mosquitto_sub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS \
  -t "phev2mqtt/reconnect/set" \
  -t "phev2mqtt/wifi/set" \
  -t "phev2mqtt/service/restart" \
  -t "phev2mqtt/hardreset/set" \
  -t "phev2mqtt/reboot/set" \
  -t "phev2mqtt/rollback/set" | while read line; do
  case "$line" in
    reconnect) /usr/local/bin/phev-reconnect.sh ;;
    ON)  /usr/local/bin/phev-wifi-control.sh ON ;;
    OFF) /usr/local/bin/phev-wifi-control.sh OFF ;;
    restart)
      mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS \
        -t "phev2mqtt/service/state" -m "restarting"
      systemctl restart phev2mqtt
      sleep 5
      /usr/local/bin/phev-wifi-status.sh
      ;;
    hardreset) /usr/local/bin/phev-hardreset.sh ;;
    reboot)
      mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS \
        -t "phev2mqtt/service/state" -m "rebooting"
      systemctl reboot
      ;;
    rollback) /usr/local/bin/phev-rollback.sh ;;
  esac
done
```

```bash
chmod +x /usr/local/bin/phev-mqtt-listener.sh
```

> **Why a dedicated restart command?** The WiFi reconnect button restarts `wpa_supplicant` and the network connection to the car. However, if the phev2mqtt process itself has a stale TCP session to the car (connection appears up but commands like heating are silently ignored), restarting WiFi alone will not fix it. The `restart` command targets the phev2mqtt process directly, which forces a clean TCP reconnect without disrupting the WiFi association.
>
> **Escalation order:** try **Restart Service** first (fixes a stale phev2mqtt TCP session), then **WiFi Driver Reload** (fixes a stuck WiFi radio/firmware that a service restart can't reach — it briefly drops and automatically re-establishes the connection, no settings are changed or lost), then **Reboot VM** as the last resort (forces a full USB re-enumeration). The listener service itself runs as root via systemd, so `systemctl reboot` and `modprobe` need no extra sudoers configuration.
>
> **Rollback** is not part of the escalation: it switches phev2mqtt back to the build that ran before your updates, for when a new build misbehaves. It needs the scripts from [Section 16](#16-how-to-update-and-roll-back-phev2mqtt).

### Listener systemd service

```bash
nano /etc/systemd/system/phev-reconnect-listener.service
```

```ini
[Unit]
Description=Listen for PHEV commands via MQTT
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/phev-mqtt-listener.sh
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable phev-reconnect-listener
systemctl start phev-reconnect-listener
```

### Status timer — publishes stats every 30 seconds

Create the service:

```bash
nano /etc/systemd/system/phev-wifi-status.service
```

```ini
[Unit]
Description=Publish PHEV WiFi status to MQTT

[Service]
Type=oneshot
ExecStart=/usr/local/bin/phev-wifi-status.sh
```

Create the timer:

```bash
nano /etc/systemd/system/phev-wifi-status.timer
```

```ini
[Unit]
Description=Run PHEV WiFi status every 30 seconds

[Timer]
OnBootSec=30s
OnUnitActiveSec=30s

[Install]
WantedBy=timers.target
```

```bash
systemctl daemon-reload
systemctl enable phev-wifi-status.timer
systemctl start phev-wifi-status.timer
```

### Add MQTT sensors, switch and buttons to Home Assistant configuration.yaml

```yaml
mqtt:
  button:
    - name: "PHEV Reconnect"
      command_topic: "phev2mqtt/reconnect/set"
      payload_press: "reconnect"
      unique_id: phev_reconnect_button

    - name: "PHEV Restart Service"
      command_topic: "phev2mqtt/service/restart"
      payload_press: "restart"
      unique_id: phev_service_restart_button
      icon: mdi:restart

    - name: "PHEV WiFi Driver Reload"
      command_topic: "phev2mqtt/hardreset/set"
      payload_press: "hardreset"
      unique_id: phev_hardreset_button
      icon: mdi:wifi-cog

    - name: "PHEV Reboot VM"
      command_topic: "phev2mqtt/reboot/set"
      payload_press: "reboot"
      unique_id: phev_reboot_button
      icon: mdi:restart-alert

    - name: "PHEV Rollback phev2mqtt"
      command_topic: "phev2mqtt/rollback/set"
      payload_press: "rollback"
      unique_id: phev_rollback_button
      icon: mdi:backup-restore

  switch:
    - name: "PHEV WiFi"
      command_topic: "phev2mqtt/wifi/set"
      state_topic: "phev2mqtt/wifi/enabled"
      payload_on: "ON"
      payload_off: "OFF"
      state_on: "ON"
      state_off: "OFF"
      unique_id: phev_wifi_switch

  sensor:
    - name: "PHEV WiFi IP"
      state_topic: "phev2mqtt/wifi/ip"
      unique_id: phev_wifi_ip
    - name: "PHEV WiFi Signal"
      state_topic: "phev2mqtt/wifi/signal"
      unit_of_measurement: "dBm"
      unique_id: phev_wifi_signal
    - name: "PHEV WiFi SSID"
      state_topic: "phev2mqtt/wifi/ssid"
      unique_id: phev_wifi_ssid
    - name: "PHEV WiFi Status"
      state_topic: "phev2mqtt/wifi/status"
      unique_id: phev_wifi_status
    - name: "PHEV Service State"
      state_topic: "phev2mqtt/service/state"
      unique_id: phev_service_state
      icon: mdi:cog-sync
    - name: "PHEV Service Uptime"
      state_topic: "phev2mqtt/service/uptime"
      unique_id: phev_service_uptime
      unit_of_measurement: "s"
      icon: mdi:timer
    - name: "PHEV Running Build"
      state_topic: "phev2mqtt/service/build"
      unique_id: phev_service_build
      icon: mdi:source-branch
```

### Add a friendly uptime template sensor to templates.yaml

If you use a separate `templates.yaml` file (included from `configuration.yaml`), add the following entry. It converts the raw uptime seconds into a human-readable format like `2h 14m`:

```yaml
- sensor:
    - name: "PHEV Service Uptime Friendly"
      unique_id: phev_service_uptime_friendly
      icon: mdi:timer
      state: >
        {% set s = states('sensor.phev_service_uptime') | int(0) %}
        {% set d = (s // 86400) %}
        {% set h = (s % 86400) // 3600 %}
        {% set m = (s % 3600) // 60 %}
        {% if d > 0 %}
          {{ d }}d {{ h }}h {{ m }}m
        {% elif h > 0 %}
          {{ h }}h {{ m }}m
        {% else %}
          {{ m }}m
        {% endif %}
```

Restart Home Assistant after editing both files.

---

## 15. Home Assistant Lovelace Card

Add a new card to your dashboard. Go to **Edit Dashboard → Add Card → Manual** and paste:

```yaml
type: vertical-stack
cards:
  - type: markdown
    content: "## 🚗 PHEV Gateway"

  - type: entities
    title: WiFi Connection
    icon: mdi:wifi
    entities:
      - entity: sensor.phev_wifi_status
        name: Interface Status
        icon: mdi:lan-connect
      - entity: sensor.phev_wifi_ssid
        name: Connected SSID
        icon: mdi:wifi-settings
      - entity: sensor.phev_wifi_ip
        name: IP Address
        icon: mdi:ip-network
      - entity: sensor.phev_wifi_signal
        name: Signal Strength
        icon: mdi:signal

  - type: entities
    title: phev2mqtt Service
    icon: mdi:cog
    entities:
      - entity: sensor.phev_service_state
        name: Service State
        icon: mdi:cog-sync
      - entity: sensor.phev_service_uptime_friendly
        name: Uptime
        icon: mdi:timer
      - entity: sensor.phev_running_build
        name: Running Build
        icon: mdi:source-branch

  - type: horizontal-stack
    cards:
      - type: button
        name: Reconnect WiFi
        icon: mdi:wifi-refresh
        tap_action:
          action: call-service
          service: mqtt.publish
          service_data:
            topic: phev2mqtt/reconnect/set
            payload: reconnect
        icon_height: 40px

      - type: button
        entity: switch.phev_wifi
        name: WiFi Enable
        icon: mdi:wifi
        tap_action:
          action: toggle
        show_state: true
        icon_height: 40px

      - type: button
        name: Restart Service
        icon: mdi:restart
        tap_action:
          action: call-service
          service: mqtt.publish
          service_data:
            topic: phev2mqtt/service/restart
            payload: restart
        icon_height: 40px

  - type: horizontal-stack
    cards:
      - type: button
        name: WiFi Driver Reload
        icon: mdi:wifi-cog
        tap_action:
          action: call-service
          service: mqtt.publish
          service_data:
            topic: phev2mqtt/hardreset/set
            payload: hardreset
          confirmation:
            text: "Reloads the WiFi driver and automatically reconnects (~20s). Does not change any WiFi settings."
        icon_height: 40px

      - type: button
        name: Reboot VM
        icon: mdi:restart-alert
        tap_action:
          action: call-service
          service: mqtt.publish
          service_data:
            topic: phev2mqtt/reboot/set
            payload: reboot
          confirmation:
            text: "Reboot the whole PHEV gateway VM? Takes about a minute."
        icon_height: 40px

      - type: button
        name: Rollback
        icon: mdi:backup-restore
        tap_action:
          action: call-service
          service: mqtt.publish
          service_data:
            topic: phev2mqtt/rollback/set
            payload: rollback
          confirmation:
            text: "Switch phev2mqtt back to the build that ran before your updates (phev2mqtt-stable)?"
        icon_height: 40px
```

> **When to use each button:**
> - **Reconnect WiFi** — use when the WiFi connection to the car has dropped (signal lost, car moved out of range and back)
> - **WiFi Enable/Disable** — use when you need to temporarily connect another device (phone, laptop) directly to the car's hotspot
> - **Restart Service** — use when the car is connected and WiFi sensors show healthy, but commands like heating or AC are not responding. This restarts the phev2mqtt process and forces a clean TCP reconnect to the car without touching the WiFi
> - **WiFi Driver Reload** — use when Restart Service doesn't bring commands back. Reloads the WiFi driver to clear a radio stuck in Link Power Save mode, without rebooting the VM. This does not erase or change any saved WiFi settings — it briefly drops and automatically re-establishes the same connection to the car
> - **Reboot VM** — last resort, when WiFi Driver Reload doesn't help either. Full VM reboot, forces a complete USB re-enumeration
> - **Rollback** — use when a newly installed phev2mqtt build misbehaves. Switches back to `phev2mqtt-stable` and removes the Home Assistant entities only the newer build creates (see [Section 16](#16-how-to-update-and-roll-back-phev2mqtt))

---

## 16. How to Update and Roll Back phev2mqtt

Every build is kept as its own file, `/usr/local/bin/phev2mqtt-<version>`, and `/usr/local/bin/phev2mqtt` becomes a link to the one in use. The build that was running before your first update with this section is kept as `phev2mqtt-stable`. Rolling back points the link at it again. The service file from Section 10 never changes.

### Before every update

- Take a Proxmox snapshot of the VM.
- Take a Home Assistant backup (**Settings → System → Backups**).

### Point your clone at the fork (once)

If you cloned [buxtronix/phev2mqtt](https://github.com/buxtronix/phev2mqtt) before this guide switched to the fork:

```bash
cd ~/phev2mqtt
git remote set-url origin https://github.com/corautem/phev2mqtt.git
git fetch --tags origin
```

### Install script

```bash
nano /usr/local/bin/phev-install.sh
```

```bash
#!/bin/bash
# Builds a phev2mqtt version, keeps it next to the earlier builds and
# switches the service to it.
# Usage: phev-install.sh [branch, tag or commit]   (default: master)
set -euo pipefail
export PATH="$PATH:/usr/local/go/bin"

SRC="/root/phev2mqtt"   # where you cloned phev2mqtt in Section 8
BIN="/usr/local/bin"
REF="${1:-master}"

cd "$SRC"
git fetch --tags origin
git checkout --quiet "$REF"
# For a branch, bring it up to date with GitHub.
if git show-ref --verify --quiet "refs/remotes/origin/$REF"; then
  git merge --quiet --ff-only "origin/$REF"
fi

# Named after what you asked for plus the commit, e.g. master-1a2b3c4
VER="$(echo "$REF" | tr '/' '-')-$(git rev-parse --short HEAD)"
go build -o "$BIN/phev2mqtt-$VER.new" .
mv "$BIN/phev2mqtt-$VER.new" "$BIN/phev2mqtt-$VER"

# The first time, keep the build that is running today as phev2mqtt-stable.
if [ -f "$BIN/phev2mqtt" ] && [ ! -L "$BIN/phev2mqtt" ]; then
  if [ ! -e "$BIN/phev2mqtt-stable" ]; then
    cp -p "$BIN/phev2mqtt" "$BIN/phev2mqtt-stable"
  else
    cp -p "$BIN/phev2mqtt" "$BIN/phev2mqtt-replaced-$(date +%Y%m%d-%H%M)"
  fi
fi

ln -sfn "$BIN/phev2mqtt-$VER" "$BIN/phev2mqtt"
systemctl restart phev2mqtt
logger -t phev-install "phev2mqtt now runs phev2mqtt-$VER"
echo "phev2mqtt now runs phev2mqtt-$VER (undo with phev-rollback.sh)"
```

```bash
chmod +x /usr/local/bin/phev-install.sh
```

### Update

```bash
phev-install.sh                       # latest master from GitHub
phev-install.sh BRANCH_TAG_OR_COMMIT  # or a specific version
journalctl -f -u phev2mqtt -o cat
```

You should see `%PHEV_TCP_CONNECTED%` in the logs. The **Running Build** sensor (Section 14) shows which build is in use, and `ls -l /usr/local/bin/phev2mqtt*` lists the installed builds.

### Rollback script

```bash
nano /usr/local/bin/phev-rollback.sh
```

```bash
#!/bin/bash
# Switches phev2mqtt back to phev2mqtt-stable, the build that ran before the
# first update with phev-install.sh, and removes the Home Assistant entities
# that only newer builds create.

MQTT_HOST="HA_IP_ADDRESS"
MQTT_USER="YOUR_MQTT_USER"
MQTT_PASS="YOUR_MQTT_PASS"
BIN="/usr/local/bin"

if [ ! -L "$BIN/phev2mqtt" ] || [ ! -e "$BIN/phev2mqtt-stable" ]; then
  echo "Nothing to roll back: phev2mqtt was never updated with phev-install.sh"
  exit 1
fi
ACTIVE=$(readlink -f "$BIN/phev2mqtt")
if [ "$ACTIVE" = "$(readlink -f "$BIN/phev2mqtt-stable")" ]; then
  echo "Already running phev2mqtt-stable"
  exit 0
fi
mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/service/state" -m "rolling back"

# Stopped first, so it can't announce its entities again during the cleanup.
systemctl stop phev2mqtt
# Only the newer build knows which entities it added.
"$ACTIVE" --config=/dev/null client mqtt-cleanup \
  --mqtt_server "tcp://$MQTT_HOST:1883" --mqtt_username "$MQTT_USER" --mqtt_password "$MQTT_PASS" \
  || echo "Entity cleanup failed or not supported by this build, continuing"
ln -sfn "$BIN/phev2mqtt-stable" "$BIN/phev2mqtt"
systemctl start phev2mqtt

logger -t phev-rollback "Rolled back from $(basename "$ACTIVE") to phev2mqtt-stable"
sleep 5
/usr/local/bin/phev-wifi-status.sh
echo "phev2mqtt now runs phev2mqtt-stable"
```

```bash
chmod +x /usr/local/bin/phev-rollback.sh
```

### Roll back

Run `phev-rollback.sh`, or press **Rollback** on the PHEV Gateway card (Sections 14 and 15). To go forward again, run `phev-install.sh` with the version you want, or point the link at a build that is already installed:

```bash
ln -sfn /usr/local/bin/phev2mqtt-VERSION /usr/local/bin/phev2mqtt
systemctl restart phev2mqtt
```

> **Optional features stay switched on across a rollback.** They are set with environment variables ([Section 20](#20-optional-extra-sensors)), which `phev2mqtt-stable` ignores, so they come back when you install a newer build again. Section 20 explains how to switch them off.

> **If the VM itself is in trouble**, restore the Proxmox snapshot you took before the update.

---

## 17. Useful Diagnostic Commands

### phev2mqtt

```bash
# Check service status
systemctl status phev2mqtt

# Follow live logs
journalctl -f -u phev2mqtt -o cat

# View all logs since last start
journalctl -u phev2mqtt -o cat --no-pager

# Register client to car
/usr/local/bin/phev2mqtt client register

# Watch live car data (debug mode, does not send to MQTT)
/usr/local/bin/phev2mqtt client watch

# Watch with verbose debug output
/usr/local/bin/phev2mqtt client watch -v=debug

# Restart the service
systemctl restart phev2mqtt
```

### WiFi and Network

```bash
# Show all network interfaces and IPs
ip addr show

# Show routing table
ip route show

# Check WiFi interface status
ip addr show YOUR_WIFI_INTERFACE_NAME

# Scan for available WiFi networks
iw dev YOUR_WIFI_INTERFACE_NAME scan | grep SSID

# Show WiFi connection details (signal, SSID, etc.)
iw dev YOUR_WIFI_INTERFACE_NAME link

# Check WiFi power save status
iw dev YOUR_WIFI_INTERFACE_NAME get power_save

# Ping the car
ping 192.168.8.46

# Ping internet
ping 1.1.1.1

# Check wpa_supplicant status
systemctl status wpa_supplicant-phev

# Check wpa_supplicant connection status
wpa_cli -i YOUR_WIFI_INTERFACE_NAME status
```

### Systemd Services

```bash
# List all phev-related services
systemctl list-units | grep phev

# Check all running timers
systemctl list-timers

# Check listener service
systemctl status phev-reconnect-listener

# Check WiFi status timer
systemctl status phev-wifi-status.timer

# Restart all phev services at once
systemctl restart wpa_supplicant-phev phev2mqtt phev-reconnect-listener
```

### Debian System

```bash
# Check system resource usage
htop

# Check disk space
df -h

# Check memory usage
free -h

# Check running processes
ps aux

# Check system logs
journalctl -xe

# Update system packages
apt update && apt upgrade -y

# Check installed package version
dpkg -l | grep <package-name>

# Check kernel version
uname -a

# Check Debian version
cat /etc/os-release
```

### MQTT

```bash
# Test publishing a message to MQTT
mosquitto_pub -h HA_IP_ADDRESS -u YOUR_MQTT_USER -P YOUR_MQTT_PASS \
  -t test/topic -m "hello"

# Subscribe to all phev MQTT topics (monitor live)
mosquitto_sub -h HA_IP_ADDRESS -u YOUR_MQTT_USER -P YOUR_MQTT_PASS \
  -t "phev/#" -v

# Subscribe to WiFi status topics only
mosquitto_sub -h HA_IP_ADDRESS -u YOUR_MQTT_USER -P YOUR_MQTT_PASS \
  -t "phev2mqtt/wifi/#" -v

# Subscribe to service status topics only
mosquitto_sub -h HA_IP_ADDRESS -u YOUR_MQTT_USER -P YOUR_MQTT_PASS \
  -t "phev2mqtt/service/#" -v

# Manually trigger reconnect via MQTT
mosquitto_pub -h HA_IP_ADDRESS -u YOUR_MQTT_USER -P YOUR_MQTT_PASS \
  -t "phev2mqtt/reconnect/set" -m "reconnect"

# Manually restart phev2mqtt service via MQTT
mosquitto_pub -h HA_IP_ADDRESS -u YOUR_MQTT_USER -P YOUR_MQTT_PASS \
  -t "phev2mqtt/service/restart" -m "restart"

# Manually turn off WiFi via MQTT
mosquitto_pub -h HA_IP_ADDRESS -u YOUR_MQTT_USER -P YOUR_MQTT_PASS \
  -t "phev2mqtt/wifi/set" -m "OFF"

# Manually turn on WiFi via MQTT
mosquitto_pub -h HA_IP_ADDRESS -u YOUR_MQTT_USER -P YOUR_MQTT_PASS \
  -t "phev2mqtt/wifi/set" -m "ON"
```

### USB and Driver

```bash
# List USB devices
lsusb

# Check if WiFi driver is loaded
lsmod | grep 88x2bu

# Check driver kernel messages
dmesg | grep RTW

# Check DKMS status (driver rebuild on kernel update)
dkms status
```

---

## 18. Known Issues and Fixes

### WiFi driver log noise

**Symptom:** Lots of `RTW:` messages in dmesg like `Dump efuse in suspend`, `Invalid rate 0x0`, `power state unchange`.
**Cause:** Known quirks of the RTL88x2BU out-of-tree driver.
**Fix:** These are harmless. The important line is `module init ret=0` and the interface appearing in `ip link show`.

### "Failed to query local AF_VSOCK CID" on systemctl commands

**Symptom:** Running `systemctl daemon-reload`, `enable`, `start`, or similar prints a line like:

```
systemd-ssh-generator[992]: Failed to query local AF_VSOCK CID: Cannot assign requested address
```

**Cause:** Newer systemd versions include a generator that sets up SSH access over AF_VSOCK (a VM-to-host socket some hypervisor tools use for console/SSH access without networking). This is a standard Proxmox VM with no vsock device attached, so the generator fails to query one.

**Fix:** None needed — this is not an error and is unrelated to whatever systemctl command you were actually running. It doesn't affect phev2mqtt, WiFi, or any service in this guide. Safe to ignore.

### Heating / AC commands not responding despite car being connected

**Symptom:** Car is connected and state sensors (locks, doors, battery) update normally in Home Assistant, but commands like windscreen heat or AC have no effect. Restarting the phev2mqtt service fixes it.

> **Encoding errors:** if `journalctl -u phev2mqtt -o cat | grep "too many errors"` shows `Disconnecting due to too many errors`, phev2mqtt and the car got out of step on their packet keys. Switch on `KEY_RESYNC` ([Section 20](#20-optional-extra-sensors)).

> **Check this first:** the car only runs remote climate with its power switch **off**, and accepts the command without acting on it otherwise. With the extra sensors switched on ([Section 20](#20-optional-extra-sensors)), the **Power Switch** sensor shows this. If it reads `acc` or `on`, the cause is the car, not the connection.

**Cause:** phev2mqtt maintains a persistent TCP connection to the car (`192.168.8.46:8080`). This connection can silently enter a hung state where the socket appears open but outgoing writes fail. State reads continue working because the car pushes register notifications to phev2mqtt regardless, but write commands (heating, AC, locks) require a functioning bidirectional TCP session.

This is often triggered or worsened by the WiFi adapter entering **Link Power Save (LPS)** mode — the firmware puts the radio into a low-power sleep state and gets stuck there, causing TX failures visible in dmesg as:

```
rtw_8822bu 5-1:1.0: firmware failed to leave lps state
rtw_8822bu 5-1:1.0: failed to send h2c command
```

**Fix:** Try the buttons in the Home Assistant PHEV Gateway card (see Section 15) in this order — or install the [Self-Healing Watchdog](#19-self-healing-watchdog) (Section 19) to have this done automatically:

1. **Restart Service** — forces a clean TCP reconnect to the car. Fixes it when the hang is in phev2mqtt itself (a stale TCP session) while the WiFi radio is fine.
2. **WiFi Driver Reload** — reloads the WiFi driver module. Fixes it when the radio firmware itself is stuck in LPS and won't wake up for TX, which a phev2mqtt-level restart can't reach. Briefly drops and automatically re-establishes the WiFi connection; no settings are changed.
3. **Reboot VM** — full VM reboot, forcing a complete USB re-enumeration. Use if the driver reload doesn't clear it either.

To prevent the LPS errors from occurring in the first place, make sure both power-save layers are disabled:

```bash
# 802.11 Link Power Save (see Section 5)
systemctl status wifi-powersave-off.service
iw dev YOUR_WIFI_INTERFACE_NAME get power_save
# Expected: Power save: off

# USB autosuspend (a separate layer, see Section 5)
cat /sys/module/usbcore/parameters/autosuspend
# Expected: -1
```

Both settings address different layers of the same symptom, and it's worth confirming both are correctly applied — a stuck adapter has historically recurred even with 802.11 power save disabled, which is why USB autosuspend is worth ruling out too.

> **Note:** Do not use the WiFi Reconnect button for this problem — that restarts the WiFi association but leaves the phev2mqtt process and its stale TCP session untouched, and doesn't reload the driver either. It only helps when the WiFi association itself (not the TCP session or the radio firmware) has dropped.

### wifi-powersave-off.service fails at boot with "No such device"

**Symptom:** `systemctl status wifi-powersave-off.service` shows `failed` with `iw: command failed: No such device (-19)` in the logs.

**Cause:** The service uses `After=network.target` which does not guarantee the WiFi interface exists yet. The USB WiFi adapter takes longer to enumerate than `network.target` fires, so `iw` runs before the interface is available. This timing issue became more common after Debian updated the `firmware-realtek` package, which changed how quickly the native driver initialises.

**Fix:** Replace `After=network.target` with a direct dependency on the interface's device unit:

```bash
nano /etc/systemd/system/wifi-powersave-off.service
```

```ini
[Unit]
Description=Disable WiFi power save
After=sys-subsystem-net-devices-YOUR_WIFI_INTERFACE_NAME.device
Requires=sys-subsystem-net-devices-YOUR_WIFI_INTERFACE_NAME.device

[Service]
Type=oneshot
ExecStart=/sbin/iw dev YOUR_WIFI_INTERFACE_NAME set power_save off
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl restart wifi-powersave-off.service
iw dev YOUR_WIFI_INTERFACE_NAME get power_save
# Expected: Power save: off
```

### Internet stops working when connected to PHEV WiFi

**Symptom:** `apt install` or `ping 1.1.1.1` fails after connecting to car WiFi.
**Cause:** The PHEV's DHCP adds a default route via `192.168.8.46` that takes priority over ethernet.
**Fix:** The `RouteMetric=200` in `/etc/systemd/network/20-wifi.network` prevents this permanently. If it happens manually:

```bash
ip route del default via 192.168.8.46 dev YOUR_WIFI_INTERFACE_NAME
```

### phev2mqtt broken pipe / keeps disconnecting from MQTT

**Symptom:** `write: broken pipe` in logs, service keeps restarting.
**Cause:** Another instance of phev2mqtt (e.g. on the Raspberry Pi) is connecting with the same client ID and kicking the VM off.
**Fix:** Stop and disable phev2mqtt on the Raspberry Pi:

```bash
sudo systemctl stop phev2mqtt
sudo systemctl disable phev2mqtt
```

### phev2mqtt service starts but doesn't connect to the car

**Symptom:** Service shows `active (running)` but no `%PHEV_TCP_CONNECTED%` in logs.
**Cause:** Service started before wpa_supplicant finished connecting to the PHEV WiFi.
**Fix:** The service will automatically retry every 30 seconds. If it still fails after several retries:

```bash
systemctl restart wpa_supplicant-phev
systemctl restart phev2mqtt
```

### wpa_supplicant service fails to start

**Symptom:** `Job for wpa_supplicant-phev.service failed`.
**Cause:** A manually started wpa_supplicant instance is already running for the same interface.
**Fix:**

```bash
ps aux | grep wpa_supplicant
kill PID
systemctl restart wpa_supplicant-phev
```

### DHCP not assigning 192.168.8.47

**Symptom:** WiFi connects to car but no IPv4 address appears.
**Fix:** Restart systemd-networkd to trigger DHCP:

```bash
systemctl restart systemd-networkd
```

Or install and use dhclient manually:

```bash
apt install -y isc-dhcp-client
dhclient YOUR_WIFI_INTERFACE_NAME
```

### WiFi interface disappears after kernel update

**Symptom:** After an automatic kernel update the WiFi interface (`wlx...`) disappears completely from `ip link show`, wpa_supplicant fails to start, and dmesg shows:

```
rtw_8822bu: firmware: failed to load rtw88/rtw8822b_fw.bin (-2)
rtw_8822bu: failed to load firmware
```

**Cause:** When the kernel updates from 6.1 to 6.12+, the old out-of-tree `88x2bu` DKMS driver is no longer loaded. Kernel 6.12+ includes a native in-kernel `rtw88_8822bu` driver, but it requires a firmware file that is not installed by default.

**Fix:**

1. Enable non-free-firmware repository if not already done:

```bash
nano /etc/apt/sources.list.d/debian.sources
```

Add `contrib non-free non-free-firmware` to both `Components:` lines.

2. Install the firmware package:

```bash
apt update
apt install -y firmware-realtek
```

3. Reload the driver:

```bash
modprobe -r rtw88_8822bu
modprobe rtw88_8822bu
ip link show
```

4. Make it permanent across reboots:

```bash
echo "rtw88_8822bu" >> /etc/modules
update-initramfs -u
reboot
```

After this you are running the native in-kernel driver — no more DKMS rebuilds needed on future kernel updates.

### PHEV WiFi sensors in Home Assistant show Unknown state

**Symptom:** WiFi sensors in HA show Unknown state.
**Cause:** The status script hasn't run yet or mosquitto-clients is not installed.
**Fix:**

```bash
apt install -y mosquitto-clients
chmod +x /usr/local/bin/phev-wifi-status.sh
/usr/local/bin/phev-wifi-status.sh
```

---

## 19. Self-Healing Watchdog

> **Optional but recommended.** Automates the manual escalation described in [Known Issues](#18-known-issues-and-fixes) for "Heating / AC commands not responding" — Restart Service → WiFi Driver Reload → Reboot VM — so it happens without you noticing the problem first. Requires the `phev-hardreset.sh` script from Section 14 to already be installed.

### How it works

Every 4 minutes, a timer checks the kernel log for the exact stuck-radio signature already identified in Section 18 (`firmware failed to leave lps state` / `failed to send h2c command`), looking only at log lines that appeared since its last check (so nothing is double-counted or missed at the boundary between runs). If the signature is present, it escalates based on a counter that persists across ticks but resets on recovery:

- **1st detection** → restarts phev2mqtt (equivalent to the Restart Service button)
- **2nd consecutive detection** (the restart didn't help) → runs `phev-hardreset.sh` (equivalent to WiFi Driver Reload)
- **3rd+ consecutive detection** (the driver reload didn't help either) → reboots the VM — but only if it hasn't already auto-rebooted in the last 2 hours, to avoid a reboot loop if something is persistently broken. If that cooldown is active, it logs and publishes a "needs attention" status instead of rebooting again.

Every action — including recovering back to normal and the "needs attention" case — is logged with `logger -t phev-watchdog` (view with `journalctl -t phev-watchdog`) and published to MQTT, so you can see what it did after the fact rather than it acting silently.

### Watchdog script

```bash
nano /usr/local/bin/phev-watchdog.sh
```

```bash
#!/bin/bash

MQTT_HOST="HA_IP_ADDRESS"
MQTT_USER="YOUR_MQTT_USER"
MQTT_PASS="YOUR_MQTT_PASS"

RUN_DIR="/run/phev-watchdog"
STATE_DIR="/var/lib/phev-watchdog"
mkdir -p "$RUN_DIR" "$STATE_DIR"

COUNT_FILE="$RUN_DIR/count"
LAST_CHECK_FILE="$RUN_DIR/last_check"
LAST_REBOOT_FILE="$STATE_DIR/last_reboot"

COUNT=$(cat "$COUNT_FILE" 2>/dev/null || echo 0)
SINCE=$(cat "$LAST_CHECK_FILE" 2>/dev/null || date -d '5 minutes ago' '+%Y-%m-%d %H:%M:%S')
date '+%Y-%m-%d %H:%M:%S' > "$LAST_CHECK_FILE"

STUCK=0
if journalctl -k --since "$SINCE" 2>/dev/null | grep -qE "failed to leave lps state|failed to send h2c command"; then
  STUCK=1
fi

if [ "$STUCK" = "1" ]; then
  COUNT=$((COUNT + 1))
  echo "$COUNT" > "$COUNT_FILE"

  if [ "$COUNT" -eq 1 ]; then
    logger -t phev-watchdog "Stuck radio detected, restarting phev2mqtt service"
    mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/watchdog/action" -m "restart_service"
    systemctl restart phev2mqtt

  elif [ "$COUNT" -eq 2 ]; then
    logger -t phev-watchdog "Stuck radio persists after service restart, reloading WiFi driver"
    mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/watchdog/action" -m "driver_reload"
    /usr/local/bin/phev-hardreset.sh

  else
    LAST_REBOOT=$(cat "$LAST_REBOOT_FILE" 2>/dev/null || echo 0)
    NOW=$(date +%s)
    if [ $((NOW - LAST_REBOOT)) -gt 7200 ]; then
      logger -t phev-watchdog "Stuck radio persists after driver reload, rebooting VM"
      mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/watchdog/action" -m "reboot"
      echo "$NOW" > "$LAST_REBOOT_FILE"
      systemctl reboot
    else
      logger -t phev-watchdog "Stuck radio persists but rebooted within the last 2 hours already, holding off to avoid a loop"
      mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/watchdog/action" -m "stuck_needs_attention"
    fi
  fi
else
  if [ "$COUNT" != "0" ]; then
    logger -t phev-watchdog "Recovered"
    mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/watchdog/action" -m "recovered"
  fi
  echo 0 > "$COUNT_FILE"
fi
```

```bash
chmod +x /usr/local/bin/phev-watchdog.sh
```

### Watchdog systemd service and timer

```bash
nano /etc/systemd/system/phev-watchdog.service
```

```ini
[Unit]
Description=Check for a stuck PHEV WiFi radio and self-heal

[Service]
Type=oneshot
ExecStart=/usr/local/bin/phev-watchdog.sh
```

```bash
nano /etc/systemd/system/phev-watchdog.timer
```

```ini
[Unit]
Description=Run the PHEV watchdog every 4 minutes

[Timer]
OnBootSec=4min
OnUnitActiveSec=4min

[Install]
WantedBy=timers.target
```

```bash
systemctl daemon-reload
systemctl enable phev-watchdog.timer
systemctl start phev-watchdog.timer
```

### Add a Home Assistant sensor for watchdog activity

Add to the `mqtt: sensor:` list in `configuration.yaml`:

```yaml
    - name: "PHEV Watchdog Last Action"
      state_topic: "phev2mqtt/watchdog/action"
      unique_id: phev_watchdog_action
      icon: mdi:robot-outline
```

Add it to the "phev2mqtt Service" entities card in your Lovelace dashboard (Section 15):

```yaml
      - entity: sensor.phev_watchdog_last_action
        name: Watchdog Last Action
        icon: mdi:robot-outline
```

Restart Home Assistant (or reload the MQTT integration) after editing `configuration.yaml`.

### Verifying it works

```bash
systemctl status phev-watchdog.timer
journalctl -t phev-watchdog
```

During normal operation you should see nothing alarming — the script runs silently every 4 minutes and only logs when it detects the stuck-radio signature or recovers from one. There's no safe way to manually trigger the actual LPS bug to test the escalation end-to-end; the first real confirmation will be the next time it happens naturally. When it does, `journalctl -t phev-watchdog` and the `sensor.phev_watchdog_last_action` entity will show which step it took, and heating/AC should respond again without you having touched a button.

> **Tuning:** if the watchdog escalates more aggressively than you'd like, or the 2-hour reboot cooldown is too short/long for how often this happens on your setup, the counter thresholds and cooldown value in the script can be adjusted directly.

---

## 20. Optional Extra Sensors

> **Optional.** Needs a [corautem/phev2mqtt](https://github.com/corautem/phev2mqtt) build from October 2026 or later, installed with [Section 16](#16-how-to-update-and-roll-back-phev2mqtt) so you can roll back.

phev2mqtt can publish 14 more Home Assistant entities, decoded from data the car already sends, so the car sees no extra traffic. They are off by default. The decoding is ported from [phev-remote-plus](https://github.com/Anwo91/phev-remote-plus), which checked it against a 2014 car.

| Entity | What it shows |
|---|---|
| Power Switch | `off`, `acc` or `on`. Remote climate only works with the power switch off, so check this first when climate does nothing. |
| Security Alarm | `disarmed`, `armed` or `triggered` |
| Last Alarm Event | When the alarm last went off |
| Charge Type | `none`, `normal`, `trickle` or `quick` (CHAdeMO rapid charging) |
| Waiting For Car Charge Timer | On while the car is plugged in but held back by its own charge timer. If a smart charger or your energy supplier decides when the car charges, this should stay off; if it's on, the car's timer is overruling the charger. |
| Scheduled Charge Failed | The car could not start a scheduled charge |
| Car Charge Timer | The car's own charge timer is on. The schedule is in the attributes. |
| Car Climate Timer | The car's own climate timer is on. The start times are in the attributes. |
| Battery Heater | 2014–2017 cars only: the drive battery heater is running |
| Battery Too Cold | The car asks to be plugged in to protect a cold drive battery |
| EV-ECU Error | The car reports an EV-ECU communication error (diagnostic) |
| Car Screen In Use | The car's own screen is in use; the car then ignores some remote commands |
| Registration Standby | The car is in registration mode (diagnostic) |
| Car Clock | The car's own clock, which its charge and climate timers run on (diagnostic) |

> **Car Clock and Last Alarm Event** read the car's clock in the VM's timezone. Set the VM to your local timezone ([Section 3](#3-first-steps-after-fresh-install)); on UTC they show an hour out during summer time.

### Switch them on

Options are set with environment variables in a systemd drop-in file, not with command-line flags. An older build such as `phev2mqtt-stable` ignores the variables but would refuse to start with a flag it doesn't know, so a rollback keeps working.

```bash
mkdir -p /etc/systemd/system/phev2mqtt.service.d
nano /etc/systemd/system/phev2mqtt.service.d/options.conf
```

```ini
[Service]
Environment=EXTRA_SENSORS=true
```

```bash
systemctl daemon-reload
systemctl restart phev2mqtt
```

The entities appear on the **PHEV &lt;VIN&gt;** device (**Settings → Devices & Services → MQTT**) as soon as phev2mqtt reconnects to the car, and get their values a few seconds later. **Add to dashboard** on the device page puts them on a dashboard.

### Switch them off

Set `EXTRA_SENSORS=false` in `options.conf` (or delete the file), then:

```bash
systemctl daemon-reload
systemctl restart phev2mqtt
```

phev2mqtt then removes these entities from Home Assistant. A rollback ([Section 16](#16-how-to-update-and-roll-back-phev2mqtt)) removes them too.

### Fixed in the same builds (always on)

- **Charger Connected** now reads connected while the car is plugged in inside its charge window but not charging. It used to read unplugged.
- **Interior Lights** now reads on for every value the car uses for on.
- **After a Home Assistant restart**, every phev2mqtt sensor gets its last known value again within a few seconds. The car sends most values only when they change, so they used to stay unknown until phev2mqtt restarted.
- **Phev climate state** shows the running mode (`heat`, `cool`, `windscreen`) or `off`. It used to always show `unknown`.

### More buttons (EXTRA_COMMANDS)

`Environment=EXTRA_COMMANDS=true` in `options.conf` adds these buttons to the **Controls** section:

| Button | What it does |
|---|---|
| Phev Refresh | Asks the car to send all its values |
| Phev Flash Lights | Headlights on for 5 seconds, to find the car; refused while the headlights are on |
| Phev Sync Car Clock | Sets the car's clock from the VM's (set the VM's timezone first, Section 3) |
| Phev Charge Now | Suspends the car's own charge timer, so it charges now |
| Phev Resume Car Charge Timer | Lets the car's own charge timer decide again; refused when the car has no charge timer programmed, as a timer with no charge window could stop it charging |

**Phev Last Command Result** (Diagnostic) shows how each went, for example `Flash Lights: done`, or why the car didn't take it.

### Safer climate start (CLIMATE_GUARD)

`Environment=CLIMATE_GUARD=true` makes **Phev Heat**, **Phev cool**, **Phev windscreen** and **Phev climate state** start the climate the way Mitsubishi's app does:

- On 2014–2017 cars, starting the climate keeps the car's own climate timer settings instead of overwriting them. Switching off only switches the climate off.
- A start is refused, with the reason in **Phev Last Command Result**, while the power switch is ACC or ON or the car is quick charging: the car would accept it and do nothing.
- phev2mqtt waits up to 30 seconds for the car to report the climate running, and the result says whether it did, for example `Heat 10 min: done, the climate is running`.

### Set the car's clock at every connection (SYNC_CLOCK)

`Environment=SYNC_CLOCK=true` sets the car's clock from the VM's once each time phev2mqtt connects to the car, as Mitsubishi's app does. The car's own charge and climate timers run on that clock.

### Recover from encoding errors (KEY_RESYNC)

phev2mqtt and the car scramble every packet with a key that moves on with each packet. If the two get out of step, the car rejects packets and names the key it expected. Older builds ignored that name and kept resending with the next wrong key, until phev2mqtt gave up with `Disconnecting due to too many errors` in its log, for example during a quick series of vehicle setting changes ([Section 21](#21-vehicle-settings)).

With this line in `options.conf`, phev2mqtt moves to the key the car expects, resends a rejected write at most 3 times, and asks the car for a new key if complaints pile up:

```ini
Environment=KEY_RESYNC=true
```

Its log then shows `%PHEV_KEY_RESYNC%` lines where it recovered, instead of the disconnect:

```bash
journalctl -u phev2mqtt -o cat | grep -E 'KEY_RESYNC|REKEY|SETREG_RETRY|too many errors'
```

### Good to know

- **Charger Connected with a smart charger:** while a smart charger holds off outside its charge slots, the car itself reports no charger, so Charger Connected reads unplugged even with the cable in. Your charger's own Home Assistant integration shows whether the cable is in.

---

## 21. Vehicle Settings

> **Optional.** Needs a [corautem/phev2mqtt](https://github.com/corautem/phev2mqtt) build from October 2026 or later, installed with [Section 16](#16-how-to-update-and-roll-back-phev2mqtt).

The car's own customisable functions, the ones Mitsubishi's app lists under "Vehicle settings", can be shown and changed in Home Assistant: lights, locks and unlock mode, lock beeps and horn answerback, wipers, turn signals, power windows and A/C behaviour. The car reports them itself; which ones appear depends on how your car is equipped. The decoding is ported from [phev-remote-plus](https://github.com/Anwo91/phev-remote-plus), which worked it out from Mitsubishi's app.

`VEHICLE_SETTINGS` has three values:

| Value | What you get |
|---|---|
| `off` (default) | Nothing; any vehicle settings entities are removed |
| `read` | Each setting as a read-only sensor in the **Diagnostic** section |
| `write` | Each setting as a drop-down in the **Configuration** section that changes it on the car |

Start with `read`, check a few values against the car, and only then switch to `write`.

### Switch them on

Add a line to the drop-in file from [Section 20](#20-optional-extra-sensors) (create it as shown there if you don't have it):

```bash
nano /etc/systemd/system/phev2mqtt.service.d/options.conf
```

```ini
[Service]
Environment=EXTRA_SENSORS=true
Environment=VEHICLE_SETTINGS=read
```

```bash
systemctl daemon-reload
systemctl restart phev2mqtt
```

The settings appear on the **PHEV &lt;VIN&gt;** device, named after their group, for example **Phev Lights: Headlights ON when exiting Vehicle** or **Phev Door locks: Automatic Unlock**. **Phev Car Accepts Setting Changes** shows whether the car currently accepts changes (`allowed` or `refused`).

The car sends its settings a few at a time, carrying on through its list while phev2mqtt is connected, so the list fills in over the first minutes or hours. Once a setting has appeared it stays, and phev2mqtt remembers it across restarts of phev2mqtt and Home Assistant.

The theft-alarm settings and "reset all settings" are never offered, as in Mitsubishi's own app. [entities-outlander-phev-2014-2017.md](./entities-outlander-phev-2014-2017.md) lists every entity and the settings a 2014–2017 car offers.

### Check the readings first

Pick settings you can see on the car, for example:

- **Turn Signal Lights Answerback**: count the indicator flashes when you lock and unlock with the remote. If the car is set to deadlock with one press, locking flashes three times for the deadlock instead.
- **Auto Fold Mirrors**: whether the mirrors fold when you lock.
- **Duration Dome Light Remains On after Door is Closed**: time the interior light after closing the door.

### Changing settings

Set `Environment=VEHICLE_SETTINGS=write` in `options.conf`, then `systemctl daemon-reload && systemctl restart phev2mqtt`. The read-only sensors become drop-downs offering only the options your car has.

Changes are locked by **Phev Vehicle Settings: Allow Changes**, a switch in the device's **Controls** section, so a setting can't be changed by mistake. It is off whenever phev2mqtt starts. While it's off, the drop-downs still show the car's values, but a change is refused and the drop-down goes back to what the car has. Switch it on to make changes; it switches itself off 10 minutes after you switch it on or after your last change, or switch it off yourself when you're done.

When you pick a new option, phev2mqtt does what Mitsubishi's app does: it sends the change, waits up to 8 seconds for the car to report the new value, then tells the car to save it. One change runs at a time. **Phev Vehicle Settings Last Change** shows the result, for example `Lights: Headlights ON when exiting Vehicle -> 30 seconds: done`. If the car doesn't confirm, the drop-down goes back to what the car reports, and the result says to check the setting in the car. If the connection to the car drops between the change and the save, the result says so, and phev2mqtt sends the save when the car reconnects.

phev2mqtt refuses a change while **Allow Changes** is off, while the car isn't connected, while the car reports it doesn't accept changes, or for an option your car doesn't offer.

### Snapshots and restoring

Before the first change after each start, phev2mqtt saves every known setting to a file in `/var/lib/phev2mqtt`, named like `vehicle-settings-20261015-093521.json`. A setting the car reports only later is added to that file, with its old value, before it is first changed. To put all settings back the way they were in a snapshot, with phev2mqtt running in `write` mode and connected to the car:

```bash
ls -l /var/lib/phev2mqtt/
source <(grep -E '^MQTT_(HOST|USER|PASS)=' /usr/local/bin/phev-wifi-status.sh)
phev2mqtt --config=/dev/null client restore-settings --mqtt_server tcp://$MQTT_HOST:1883 \
  --mqtt_username $MQTT_USER --mqtt_password $MQTT_PASS \
  --file /var/lib/phev2mqtt/vehicle-settings-DATE-TIME.json
```

The restore switches on **Allow Changes** itself. phev2mqtt changes the settings that differ, one at a time, and reports each in **Phev Vehicle Settings Last Change**; settings that already match show as `already set, nothing to change`. You can also change any setting back on the car's own screen.

> **Before rolling back** ([Section 16](#16-how-to-update-and-roll-back-phev2mqtt)): a rollback removes these entities, but settings you changed stay changed on the car. Restore a snapshot first if you want them back.

### Switch them off

Set `VEHICLE_SETTINGS=off` in `options.conf` (or delete the line), then:

```bash
systemctl daemon-reload
systemctl restart phev2mqtt
```

phev2mqtt then removes these entities, and the values it remembered, from Home Assistant and the broker. A rollback removes them too. Settings on the car are not changed by switching off.
