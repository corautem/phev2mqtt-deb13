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
16. [How to Update phev2mqtt](#16-how-to-update-phev2mqtt)
17. [Useful Diagnostic Commands](#17-useful-diagnostic-commands)
18. [Known Issues and Fixes](#18-known-issues-and-fixes)

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

| Chipset | Standard | In-kernel since | Notes |
|---|---|---|---|
| **Mediatek mt7921au** | WiFi 6 / AXE3000 | Kernel 5.18 (2022) | Best overall choice, very stable |
| **Mediatek mt7612u** | WiFi 5 / AC1200 | Kernel 4.19 (2018) | Reliable, widely available |
| **Mediatek mt7610u** | WiFi 5 / AC600 | Kernel 4.19 (2018) | Budget option, USB 2.0 |
| **Realtek rtl8812bu** | WiFi 5 / AC1200 | Kernel 6.2 (2023) | Kernel 6.12+ recommended |
| **Realtek rtl8812au** | WiFi 5 / AC1200 | Kernel 6.14 (2025) | Previously required out-of-tree driver |
| **Mediatek mt7601u** | WiFi 4 / N150 | Kernel 4.2 (2015) | Basic, managed mode only |

> **Recommended pick:** The **Mediatek mt7921au** chipset (e.g. Panda PAU0F, EDUP EP-AX1672) is the best choice for a new purchase — WiFi 6, very stable, plug and play on Debian 13.

> For a comprehensive and up-to-date list of compatible adapters and models, see the [morrownr/USB-WiFi](https://github.com/morrownr/USB-WiFi) repository which is the definitive reference for Linux USB WiFi adapter compatibility.

> **Note on the RTL88x2BU (this guide's adapter):** The RTL88x2BU chipset used in the TP-Link Archer T3U Plus does have in-kernel support since kernel 6.2, but kernel 6.12+ is recommended for stability. Debian 13 ships with kernel 6.1 by default, which is why the out-of-tree driver from RinCat is used in this guide. If you are running a newer kernel (6.12+) you may find the adapter works without the manual driver installation — check with `dmesg | grep rtw88` after plugging it in.

### In Proxmox UI:
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

The TP-Link Archer T3U Plus uses the **RTL88x2BU** chipset which is not included in the Linux kernel. You need to install a third-party driver.

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

To make this permanent, create a systemd service:
```bash
nano /etc/systemd/system/wifi-powersave-off.service
```
```ini
[Unit]
Description=Disable WiFi power save
After=network.target

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

> **Driver log messages explained:** After installing the driver you will see messages like `Dump efuse in suspend`, `Invalid rate 0x0`, `power state unchange` in dmesg. These are known quirks of this driver and are harmless. The important line confirming success is:
> `rtw_ndev_init(wlan0) mac_addr=xx:xx:xx:xx:xx:xx`

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
```bash
git clone https://github.com/buxtronix/phev2mqtt.git
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

> **This section is optional but recommended.** It sets up scripts that publish VM WiFi stats (IP, signal, SSID, status) to MQTT every 30 seconds, and adds the ability to reconnect or enable/disable the VM WiFi directly from Home Assistant. If you only need the car data in HA and don't need remote control of the VM WiFi, you can skip to Section 15.

These scripts publish VM WiFi stats (IP, signal, SSID, status) to MQTT every 30 seconds via a timer, and also immediately on any WiFi state change (enable, disable, reconnect). They also listen for commands from Home Assistant to reconnect or enable/disable the WiFi.

### Status script — publishes WiFi stats to MQTT
```bash
nano /usr/local/bin/phev-wifi-status.sh
```
```bash
#!/bin/bash

MQTT_HOST="HA_IP_ADDRESS"
MQTT_USER="YOUR_MQTT_USER"
MQTT_PASS="YOUR_MQTT_PASS"
IFACE="YOUR_WIFI_INTERFACE_NAME"

# Get stats
IP=$(ip addr show $IFACE | grep 'inet ' | awk '{print $2}' | cut -d/ -f1)
SIGNAL=$(iw dev $IFACE link | grep 'signal' | awk '{print $2}')
SSID=$(iw dev $IFACE link | grep 'SSID' | awk '{print $2}')
STATUS=$(cat /sys/class/net/$IFACE/operstate)
ENABLED=$([ "$(cat /sys/class/net/$IFACE/operstate 2>/dev/null)" != "down" ] && systemctl is-active --quiet wpa_supplicant-phev && echo "ON" || echo "OFF")

# Publish to MQTT
mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/wifi/ip" -m "${IP:-unavailable}"
mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/wifi/signal" -m "${SIGNAL:-unavailable}"
mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/wifi/ssid" -m "${SSID:-unavailable}"
mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/wifi/status" -m "${STATUS:-unavailable}"
mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/wifi/enabled" -m "$ENABLED"
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
    # Publish transitional state immediately so HA switch stays in sync
    mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/wifi/enabled" -m "OFF"
    ip link set $IFACE up
    systemctl start wpa_supplicant-phev
    sleep 15
    systemctl start phev2mqtt
    # Publish final state and trigger full status refresh
    mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/wifi/enabled" -m "ON"
    /usr/local/bin/phev-wifi-status.sh
    ;;
  OFF)
    # Publish OFF immediately before stopping services so HA switch stays in sync
    mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t "phev2mqtt/wifi/enabled" -m "OFF"
    systemctl stop phev2mqtt
    systemctl stop wpa_supplicant-phev
    ip link set $IFACE down
    # Trigger full status refresh so all sensors update immediately
    /usr/local/bin/phev-wifi-status.sh
    ;;
esac
```
```bash
chmod +x /usr/local/bin/phev-wifi-control.sh
```

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
  -t "phev2mqtt/wifi/set" | while read line; do
  case "$line" in
    reconnect) /usr/local/bin/phev-reconnect.sh ;;
    ON) /usr/local/bin/phev-wifi-control.sh ON ;;
    OFF) /usr/local/bin/phev-wifi-control.sh OFF ;;
  esac
done
```
```bash
chmod +x /usr/local/bin/phev-mqtt-listener.sh
```

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

### Status timer — publishes WiFi stats every 30 seconds

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

### Add MQTT sensors, switch and button to Home Assistant configuration.yaml

Once all the scripts and services above are in place, add the following to your Home Assistant `configuration.yaml` to create the sensors, switch and button entities:

```yaml
mqtt:
  button:
    - name: "PHEV Reconnect"
      command_topic: "phev2mqtt/reconnect/set"
      payload_press: "reconnect"
      unique_id: phev_reconnect_button

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
```

Restart Home Assistant after editing `configuration.yaml`.

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

  - type: horizontal-stack
    cards:
      - type: button
        name: Reconnect
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
```

---

## 16. How to Update phev2mqtt

When the [buxtronix/phev2mqtt](https://github.com/buxtronix/phev2mqtt) project receives updates on GitHub, follow these steps to update your installation:

### Step 1: Stop the running service
```bash
systemctl stop phev2mqtt
```

### Step 2: Pull the latest code
```bash
cd ~/phev2mqtt
git pull
```
This downloads the latest changes from GitHub.

### Step 3: Rebuild the binary
```bash
go build
```

### Step 4: Replace the installed binary
```bash
cp phev2mqtt /usr/local/bin/
chmod +x /usr/local/bin/phev2mqtt
```

### Step 5: Restart the service
```bash
systemctl start phev2mqtt
journalctl -f -u phev2mqtt -o cat
```

### Step 6: Verify
You should see `%PHEV_TCP_CONNECTED%` in the logs confirming the updated version is running.

> **Tip:** If something breaks after an update, you can roll back using git:
> ```bash
> systemctl stop phev2mqtt
> cd ~/phev2mqtt
> git log --oneline   # find the previous commit hash
> git checkout COMMIT_HASH
> go build
> cp phev2mqtt /usr/local/bin/
> systemctl start phev2mqtt
> ```

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

# Manually trigger reconnect via MQTT
mosquitto_pub -h HA_IP_ADDRESS -u YOUR_MQTT_USER -P YOUR_MQTT_PASS \
  -t "phev2mqtt/reconnect/set" -m "reconnect"

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
**Fix:** The service will automatically retry every 30 seconds (`RestartSec=30s`) until the WiFi is connected and the car is reachable. If it still fails to connect after several retries, manually restart the services:
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
kill PID   # replace PID with the process ID of the manual instance
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

### HA sensors show "Unknown"
**Symptom:** PHEV WiFi sensors in Home Assistant show Unknown state.
**Cause:** The status script hasn't run yet or mosquitto-clients is not installed.
**Fix:**
```bash
apt install -y mosquitto-clients
chmod +x /usr/local/bin/phev-wifi-status.sh
/usr/local/bin/phev-wifi-status.sh
```
Then check HA — sensors should update within a few seconds.
