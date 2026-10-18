# Home Assistant entities: Outlander PHEV 2014–2017

Every Home Assistant entity that [corautem/phev2mqtt](https://github.com/corautem/phev2mqtt) creates for a **2014–2017 Mitsubishi Outlander PHEV**, as found on a UK car. Cars from 2018 on talk a different protocol version and can differ.

All entities belong to the **PHEV &lt;VIN&gt;** device (**Settings → Devices & Services → MQTT**), and their names start with "Phev". The **Needs** column says which option switches an entity on ([guide Sections 20 and 21](./phev2mqtt_guide.md#20-optional-extra-sensors)):

- **always**: every build of phev2mqtt
- **extra**: `EXTRA_SENSORS=true`
- **commands**: `EXTRA_COMMANDS=true`
- **settings**: `VEHICLE_SETTINGS=read` or `write`
- **write**: `VEHICLE_SETTINGS=write`

The gateway's own entities from [guide Section 14](./phev2mqtt_guide.md#14-wifi-status-scripts-and-mqtt-sensors), such as WiFi Status and the Rollback button, are on a separate **PHEV Gateway** device.

## Controls

| Entity | Type | What it does | Needs |
|---|---|---|---|
| Phev Heat | switch | Remote heating on or off, 10 minutes | always |
| Phev cool | switch | Remote cooling on or off, 10 minutes | always |
| Phev windscreen | switch | Windscreen defrost on or off, 10 minutes | always |
| Phev climate state | select | `off`, `heat`, `cool`, `windscreen`: shows the running mode, and starts or stops it | always |
| Phev Disable Charge Timer | switch | Suspends the car's own charge timer, so it charges now | always |
| Phev Head Lights | light | Headlights on or off | always |
| Phev Park Lights | light | Parking lights on or off | always |
| Phev Restart Wifi connetion | button | Drops and reopens phev2mqtt's connection to the car (the typo is in phev2mqtt's name for it) | always |
| Phev Refresh | button | Asks the car to send all its values | commands |
| Phev Flash Lights | button | Headlights on for 5 seconds; refused while they're on | commands |
| Phev Sync Car Clock | button | Sets the car's clock from the VM's | commands |
| Phev Charge Now | button | Suspends the car's own charge timer, so it charges now | commands |
| Phev Resume Car Charge Timer | button | Lets the car's own charge timer decide again; refused when no charge window is programmed | commands |
| Phev Vehicle Settings: Allow Changes | switch | Unlocks vehicle setting changes; locks again 10 minutes after the last change, and at every start | write |

Remote climate only runs with the car's power switch **off**. With `CLIMATE_GUARD=true`, phev2mqtt refuses a start the car would ignore, keeps the car's own climate timer settings, and reports whether the climate started.

## Sensors

| Entity | Shows | Needs |
|---|---|---|
| Phev Battery | Drive battery level, % | always |
| Phev Charging | Charging / Not charging | always |
| Phev Charge Remaining | Minutes left while charging, 0 otherwise | always |
| Phev Charger Connected | Plugged in / Unplugged. This is the car's view: with a smart charger holding off between charge slots, the car reports Unplugged even with the cable in | always |
| Phev Locked | Locked / Unlocked | always |
| Phev Driver Door, Front Passenger Door, Rear Left Door, Rear Right Door | Open / Closed | always |
| Phev Bonnet, Phev Boot | Open / Closed | always |
| Phev Hazard Lights, Phev Interior Lights | Light / No light | always |
| Phev Power Switch | `off`, `acc` or `on` | extra |
| Phev Security Alarm | `disarmed`, `armed` or `triggered`. After locking, the car arms the alarm in two steps; both read `armed` | extra |
| Phev Last Alarm Event | When the alarm last went off; Unknown if it never has | extra |
| Phev Charge Type | `none`, `normal`, `trickle` or `quick` (CHAdeMO rapid charging) | extra |
| Phev Waiting For Car Charge Timer | On while plugged in but held back by the car's own charge timer | extra |
| Phev Scheduled Charge Failed | Problem when the car couldn't start a scheduled charge | extra |
| Phev Car Charge Timer | On when the car's own charge timer is on; the schedule is in the attributes | extra |
| Phev Car Climate Timer | On when the car's own climate timer is on; the start times are in the attributes | extra |
| Phev Battery Heater | Hot while the drive battery heater runs, Normal otherwise | extra |
| Phev Battery Too Cold | Problem when the car asks to be plugged in to protect a cold drive battery | extra |
| Phev Car Screen In Use | On while the car's own screen is in use; the car then ignores some remote commands | extra |

## Diagnostic

| Entity | Shows | Needs |
|---|---|---|
| Phev Last Command Result | The result of the last button press or guarded climate start, with its time | commands |
| Phev Car Clock | The car's clock as it last reported it, at the start of a session. Read in the VM's timezone | extra |
| Phev EV-ECU Error | Problem on an EV-ECU communication error | extra |
| Phev Registration Standby | On while the car is in phone registration mode | extra |
| Phev Car Accepts Setting Changes | `allowed`, or `refused` while the car won't take setting changes | settings |
| Phev Vehicle Settings Last Change | The result of the last setting change, with its time | write |

With `VEHICLE_SETTINGS=read`, the vehicle settings below also appear here, as read-only sensors.

## Vehicle settings (Configuration)

With `VEHICLE_SETTINGS=write`, each setting the car reports is a drop-down in the **Configuration** section, named after its group, for example **Phev Lights: Headlights ON when exiting Vehicle**. The car offers only some of the options Mitsubishi's catalogue has; these are the 29 settings and the options the UK car offered. Other trim levels can differ. The names and options are as Mitsubishi's app shows them, from phev2mqtt's settings catalogue, which is adapted from PHEV Link ([Credits](./README.md#credits)).

**Keyless entry & operation**

| Setting | Options |
|---|---|
| Turn Signal Lights Answerback | Lock: once, unlock: twice / Lock: once, unlock: off / Lock: off, unlock: twice / Lock: twice, unlock: once / Lock: off, unlock: once / Lock: twice, unlock: off / Off |
| Multi Mode Keyless Entry System | Off / Door Mirror: Open & Close |
| Time for Remote Unlock Inactivation after Locking | Off / 3 seconds / 5 seconds |
| The Number of Lock Button Pushes for Dead Lock | 2 times / 1 time |

**Wipers**

| Setting | Options |
|---|---|
| Windshield Wipers Intermittent Operation | Fixed (4 seconds) / Adjustable / Adjustable & Speed Sensitive / Adjustable & Rain Sensitive |
| Wipers Linked to Washer | Off / On / On with Delayed Finishing Wipe |
| Comfort Washer | Off / On |
| Rear Wiper Intermittent Interval | 0 seconds / 4 seconds / 8 seconds / 16 seconds |
| Rear Wiper Activated when in Reverse | When Rear Wiper is ON / When Front or Rear Wiper is ON |

**Lights**

| Setting | Options |
|---|---|
| Headlight Auto-cutout Function | Off / On |
| Sensitivity for Auto Light | Early / Somewhat Early / Normal / Somewhat Late / Late |
| Headlights ON when exiting Vehicle | Off / 15 seconds / 30 seconds / 1 minute / 3 minutes |
| Exterior Lights ON with Remote Unlock | Off / Parking lights / Head lights |
| Interior Light Auto-cutout Time | Off / 3 minutes / 30 minutes / 60 minutes |
| Duration Dome Light Remains On after Door is Closed | 0 seconds (no timer function) / 7.5 seconds / 15 seconds / 30 seconds / 1 minute / 2 minutes / 3 minutes |
| Charging Lid Light Auto cut-out | Off / 1 minute / 3 minutes / 5 minutes / 10 minutes |

**Turn signals**

| Setting | Options |
|---|---|
| Operation in Key Position | Ignition Switch On or Accessory / Ignition Switch On |
| Lane-change Signals (Flash Three Times with 1 Touch) | Off / On |
| Activation Time for Lane-change Signals | Short / Long |

**Door locks**

| Setting | Options |
|---|---|
| Automatic Relocking after Unlocked by Remote | 30 seconds / 1 minute / 2 minutes / 3 minutes |
| Automatic Unlock | Off / When Ignition turned to Lock |

**Air conditioning**

| Setting | Options |
|---|---|
| Recirculation Control | Manual / Automatic |
| A/C Switch Control | Manual / Automatic |
| Eco Mode | Eco / Comfort |
| Foot / Defroster Airflow Ratio | Normal / More Airflow to Foot / More Airflow to Defroster |
| Face / Foot Airflow Ratio | Normal / More Airflow to Face / More Airflow to Foot |
| Automatic Rear Defrost at Start | Off / On |

**Other**

| Setting | Options |
|---|---|
| Auto Fold Mirrors | Off / Open by Vehicle Speed / Open/Close by IG On/Off and Driver Door Open/Close / Open/Close by Keyless Entry/Operation Lock/Unlock |
| Auto Cut of ACC Power | Off / 30 minutes / 60 minutes |

The car sends its settings a few at a time, so the full list takes a while to appear after first switching this on.

### Not reported by the UK car

- Keyless entry & operation: Horn Answerback Sounds at Keyless Entry Lock
- Keyless entry & operation: Duration of Horn Answerback Sounds
- Keyless entry & operation: Keyless Operation Answerback Beep Sounds
- Lights: Auto Light Linked to Wiper
- Power windows: Timer Function after Ignition Switch Off
- Power windows: Open Operation after Ignition Switch Off
- Power windows: Power Window Position Prohibited by Lock-switch
- Door locks: Unlock Operation
- Air conditioning: Start A/C or Heater at Door Unlock
- Air conditioning: In Foot Mode, Slight Airflow through Defroster
- Air conditioning: Fast Idle for Higher Heater Performance when Stopped
- Other: Horn Answerback Sounds at Remote Engine Start

The theft-alarm settings are never shown, as in Mitsubishi's own app.

## Model notes

- **Power switch on**: the car turns its Wi-Fi off shortly after the power switch goes to ON (about 40 seconds in a 2014 car's log from [PHEV Link](https://github.com/Anwo91/phev-remote-plus) (phev-remote-plus); ACC is fine), so values stop updating while driving and come back once the car is switched off.
- **Deadlock with one press**: if "The Number of Lock Button Pushes for Dead Lock" is set to 1 time, locking with the remote flashes the indicators three times for the deadlock, whatever "Turn Signal Lights Answerback" says.
- **Values after a restart**: the car sends most values only when they change. phev2mqtt sends the last known values again after Home Assistant restarts.
