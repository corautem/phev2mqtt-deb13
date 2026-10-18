# Example automations

Home Assistant automations for a car connected with [corautem/phev2mqtt](https://github.com/corautem/phev2mqtt) as in [the guide](../phev2mqtt_guide.md), to copy and adapt.

| Automation | What it does | Needs |
|---|---|---|
| [lock-reminder.yaml](./lock-reminder.yaml) | Reminds you when the car has been left unlocked, until it's locked | |
| [climate-started-stopped.yaml](./climate-started-stopped.yaml) | Tells you when the climate starts or stops, in any mode | |
| [command-didnt-work.yaml](./command-didnt-work.yaml) | Tells you when the car refused a command or didn't start the climate, with the reason | `EXTRA_COMMANDS` or `CLIMATE_GUARD` ([guide Section 20](../phev2mqtt_guide.md#20-optional-extra-sensors)) |
| [windscreen-by-voice.yaml](./windscreen-by-voice.yaml) | Heats the windscreen for 10, 20 or 30 minutes when you ask Assist | Assist |
| [charging-light.yaml](./charging-light.yaml) | A light turns green while the car charges | A colour light |
| [gateway-needs-attention.yaml](./gateway-needs-attention.yaml) | Tells you when phev2mqtt stops or the watchdog needs help | [Guide Sections 14](../phev2mqtt_guide.md#14-wifi-status-scripts-and-mqtt-sensors) and [19](../phev2mqtt_guide.md#19-self-healing-watchdog) |

## How to add one

1. Open the file and copy all of it. On GitHub, the **Copy raw file** button above the file does that.
2. In Home Assistant, go to **Settings → Automations & Scenes → Create automation → Create new automation**, open **⋮ → Edit in YAML** and paste.
3. Change the lines marked `REPLACE`. Or switch to **Edit in visual editor** and pick the entities from the lists.
4. Save. The comments at the top of the file are dropped on saving; they're only there to explain it.

## What to fill in

- **Car entities:** their IDs contain the car's VIN, for example `binary_sensor.phev_YOUR_VIN_phev_locked`, and can look different if you renamed the device. Find yours under **Settings → Devices & Services → MQTT → PHEV &lt;VIN&gt;**: open an entity, then its cog icon. [entities-outlander-phev-2014-2017.md](../entities-outlander-phev-2014-2017.md) lists them all.
- **Phones:** with the Home Assistant Companion app, each phone has an action like `notify.mobile_app_YOUR_PHONE`. **Settings → Tools → Actions** lists them when you type "notify" (Tools was called Developer tools before Home Assistant 2026.8).
- **Speakers (optional):** Assist speakers such as Home Assistant Voice are `assist_satellite.…` entities. Leave `announce_on: []` empty to skip them.

The car only reports while it's in WiFi range of the VM, so its entities go unavailable when you drive away. These automations treat unavailable as "nothing to report".
