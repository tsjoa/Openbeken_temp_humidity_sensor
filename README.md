# Tuya Temperature & Humidity Sensor (BK7231N / SHT30) - OpenBeken Guide

Comprehensive documentation for flashing, pinout configuration, Home Assistant MQTT integration, and battery deep-sleep optimization for the **Tuya Generic Temperature and Humidity Sensor (v1.1.17)** based on the **Beken BK7231N (CBU)** microcontroller.

---

## 1. Hardware Specifications

| Component | Detail |
| :--- | :--- |
| **Microcontroller** | Beken BK7231N (Tuya CBU Module) |
| **Flash Memory** | 2048 KiB (2 MB) |
| **Sensor IC** | Sensirion SHT30 / SHT3x (I2C) |
| **Power Supply** | 2x AAA Batteries (2.2V min to 3.0V max) |
| **Status LED** | Red LED on **P26** (Active-High: 0V = OFF, 3.3V = ON) |
| **Pair / Wake Button**| Momentary Tactile Switch on **P20** (Verified active-low GPIO) |
| **Sensor Power Switch**| Transistor switch on **P17** (Active-High: 3.3V powers the I2C bus & ADC divider) |
| **Battery ADC** | Resistor divider connected to **P23 (ADC3)** |
| **Profile Slug** | `tuya-generic-temperature-and-humidity-sensor-v1.1.17` |

---
## 2. Firmware Flashing: Migrating from ESPHome-Kickstart to OpenBeken

### Why Raw `.bin` / `.rbl` Uploads Failed
When a device is running **ESPHome-Kickstart (LibreTiny `generic-bk7231n-qfn32-tuya`)**, its OTA web server (`POST /update`) strictly requires a **`.uf2`** container tagged with LibreTiny board headers and mapped to the Beken download partition (`0x12A000`). Uploading raw `.bin` or `.rbl` files fails because the LibreTiny bootloader cannot validate the partition mapping.

### Step-by-Step Migration Process

#### Step 1: Install `ltchiptool`
```bash
pipx install ltchiptool
# Or: pip install ltchiptool
```

#### Step 2: Download OpenBK7231N Release Binary
Download the `OpenBK7231N` `.rbl` binary from [OpenBK7231T_App Releases](https://github.com/openshwprojects/OpenBK7231T_App/releases):
```bash
curl -LO https://github.com/openshwprojects/OpenBK7231T_App/releases/download/1.18.301/OpenBK7231N_1.18.301.rbl
```

#### Step 3: Package RBL into LibreTiny-Compatible `.uf2`
Convert the RBL into a UF2 binary targeting the Tuya BK7231N board:
```bash
ltchiptool uf2 write \
  -b generic-bk7231n-qfn32-tuya \
  -o OpenBK7231N_1.18.301.uf2 \
  "OpenBK7231N_1.18.301.rbl=device:download"
```

#### Step 4: Flash via Kickstart Web OTA
Upload the generated UF2 to the ESPHome Kickstart web endpoint:
```bash
curl -F "update=@OpenBK7231N_1.18.301.uf2" http://192.168.20.20/update
```
The device reboots, and the Beken bootloader unpacks OpenBeken into the active application partition.

#### Step 5: Upgrade to OpenBeken "Sensors" Build
OpenBeken's standard build does not include the full `SHT3X` driver. Once running OpenBeken, upgrade to the sensors build via OpenBeken's `/api/ota` endpoint:
```bash
curl -LO https://github.com/openshwprojects/OpenBK7231T_App/releases/download/1.18.301/OpenBK7231N_1.18.301_sensors.rbl
curl -X POST --data-binary @OpenBK7231N_1.18.301_sensors.rbl http://192.168.20.20/api/ota
curl -s "http://192.168.20.20/index?restart=1"
```

---

## 3. OpenBeken Pinout & Channel Mapping

The hardware pinout extracted from the factory Tuya device profile (`tuya-generic-temperature-and-humidity-sensor-v1.1.17`):
| Pin | OpenBeken Role | Role ID | Channels | Description |
| :--- | :--- | :--- | :--- | :--- |
| **P7** | `SHT3X_SCK` | 49 | — | I2C Clock |
| **P8** | `SHT3X_SDA` | 48 | **Ch 1, Ch 2** | I2C Data (Channel 1 = Temp, Channel 2 = Humidity) |
| **P17**| `AlwaysHigh` | 34 | — | Sensor & Battery ADC Power Rail Switch |
| **P20**| `Btn_n` / `DoorSnsrWSleep` | 4 / 58 | **Ch 0** | Physical Button (Verified on P20) |
| **P23**| `BAT_ADC` | 60 | **Ch 3** | Battery Voltage ADC (linked to Channel 3) |
| **P26**| `AlwaysLow` | 35 | — | Red Status LED (0V = OFF, prevents parasitic drain) |
| *All others* | ` ` (None) | 0 | — | Unassigned / High Impedance |

### Channel Types
* **Channel 1**: `Temperature_div10` (interprets raw `267` as `26.7 °C`)
* **Channel 2**: `Humidity` (interprets raw `36` as `36 %`)
* **Channel 3**: `Voltage` (Battery ADC voltage)

### OpenBeken CLI Configuration Commands
# Pin roles and channels
SetPinRole 7 SHT3X_SCK
SetPinRole 8 SHT3X_SDA
SetPinChannel 8 1 2
SetPinRole 17 AlwaysHigh
SetPinRole 20 DoorSnsrWSleep
SetPinRole 23 BAT_ADC
SetPinChannel 23 3
SetPinRole 26 AlwaysLow

# Channel formatting
SetChannelType 1 Temperature_div10
SetChannelType 2 Humidity
SetChannelType 3 Voltage

# Persist to flash
save
```

---

## 4. Home Assistant & MQTT Configuration

### MQTT Broker Settings
* **Host**: `192.168.20.226` (Port: `1883`)
* **User**: `addons`
* **Client Topic**: `tuya_temp_hum`
* **Group Topic**: `bekens_n`

### OpenBeken Commands:
```bash
mqtt_host 192.168.20.226
mqtt_port 1883
mqtt_user addons
mqtt_pass <MQTT_PASSWORD>
mqtt_client tuya_temp_hum
mqtt_group bekens_n
save
```

### Triggering Home Assistant Auto-Discovery
```bash
curl -s "http://192.168.20.20/ha_discovery?prefix=homeassistant"
```

### Registered Home Assistant Entities (`device: obk55A19113`)

| Entity ID | Name | Type | Function |
| :--- | :--- | :--- | :--- |
| `switch.obk55a19113_stay_awake` | **Stay Awake** | Switch | Toggles between Deep Sleep (OFF) and Continuous Awake Mode (ON) |
| `sensor.obk55a19113_temperature` | Temperature | Sensor | Ambient Temperature (SHT30) in `°C` |
| `sensor.obk55a19113_humidity` | Humidity | Sensor | Ambient Humidity (SHT30) in `%` |
| `sensor.obk55a19113_voltage` | Voltage | Sensor | Battery ADC Voltage in `mV` |
| `sensor.obk55a19113_battery` | Battery | Sensor | Battery Level in `%` |
| `sensor.obk55a19113_temperature_3`| Temperature (Diag) | Sensor | Internal SoC Die Temperature in `°C` |
| `sensor.obk55a19113_rssi` | RSSI | Sensor | Wi-Fi Signal Strength in `dBm` |
| `sensor.obk55a19113_uptime` | Uptime | Sensor | System Uptime in `s` |
| `sensor.obk55a19113_ip` | IP | Sensor | Device IP Address |

### Why Entities Become "Unavailable"
By default, Home Assistant tracks device online status using MQTT's **Last Will and Testament (LWT)** on the availability topic (`tuya_temp_hum/connected`). When the sensor enters deep sleep, its TCP connection closes, causing Mosquitto to broadcast `offline`. Home Assistant then greys out the sensor cards and marks them as **"Unavailable"** until the next wake-up.

### The Fix: OpenBeken Flag 35 (Omit Availability Topic)
Enabling **Flag 35** instructs OpenBeken to omit `availability_topic` (`avty_t`) from Home Assistant Auto-Discovery. Home Assistant will then **permanently display the last received values** (temperature, humidity, voltage, battery) on dashboard cards without flipping to "Unavailable".
---

## 6. Power Consumption, Thermals, and Deep Sleep

### The Always-On Problem (Why the chip feels warm)
* In **Always-On** mode, the Wi-Fi transceiver and CPU run 24/7, drawing **80–100 mA** continuous current ($~0.3\text{ W}$).
### Deep Sleep Solution (6–12+ Months Battery Life)
In **Deep Sleep**, the SoC disables its radio and clocks, dropping power consumption to **$\approx 20\text{–}30\ \mu\text{A}$**. The device remains completely cold (room temperature) and wakes only for 1.5–2 seconds per cycle to transmit data.

### Production `autoexec.bat` / Startup Script (with Bidirectional Switch Sync)
Set this script in OpenBeken (**Config $\rightarrow$ Change Startup Command Text**):

```batch
; Enable low power 802.11 modem sleep
PowerSave 1

; Optimize MQTT and WiFi quick connect
SetFlag 35 1
SetFlag 7 1
SetFlag 37 1

; Start drivers
startDriver SHT3X
startDriver Battery
Battery_Setup 2000 3000 2.29 2400 4096

; Link physical Pin 20 button directly to Channel 5 (Stay Awake switch)
addEventHandler OnClick 20 "toggleChannel 5"
addEventHandler OnHold 20 "toggleChannel 5"
DSEdge 1 20

; Wait for Wi-Fi and MQTT connection
waitFor WiFiState 4
waitFor MQTTState 1

; Capture fresh sensor and battery readings
battery_measure
SHT_Measure
delay_ms 250

; Publish channel data to MQTT
publishChannels

; Allow TCP/MQTT network buffer to flush over RF before powering off radio
delay_s 2

; If 'Stay Awake' switch in Home Assistant is OFF (Channel 5 == 0), enter Deep Sleep for 10 min
if $CH5==0 then PinDeepSleep 600
```

### Critical Timing & Buffer Pitfalls

#### Why Auto-Wake Temperature Was Stale (The Network TX Race Condition)
* **The Symptom**: When waking up automatically every 10 minutes, the sensor reported the exact same temperature as the previous cycle, but pressing the button yielded a fresh reading.
* **The Cause**: `publishChannels` enqueues MQTT messages into lwIP's TCP transmit buffer asynchronously. Without a delay, `PinDeepSleep` was executed microseconds later, powering off the 2.4GHz RF transceiver before the TCP packet left the device over Wi-Fi. Mosquitto never received the new packet, leaving Home Assistant displaying the last retained value. Pressing the physical button triggered `toggleChannel 5` (`$CH5 == 1`), skipping deep sleep and keeping the radio active long enough to flush the queue.
* **The Fix**:
  1. `battery_measure` & `SHT_Measure` followed by `delay_ms 250` ensure the I2C measurement and ADC conversion are complete before values are formatted.
  2. `publishChannels` followed by `delay_s 2` ensures the full TCP packet exchange (Wi-Fi frame transmission and MQTT PUBACK) completes before the radio is powered down.

### Operating Procedure (How to Switch Between Deep Sleep and Awake Modes)

Because the sensor is in low-power deep sleep for 99.7% of the time, mode switching is controlled via the Home Assistant switch and a simple battery power-cycle:

#### 1. Normal Battery Operation (Deep Sleep Mode - 6 to 12+ Months Battery Life):
1. In Home Assistant, ensure the **"Stay Awake"** switch is set to **`OFF`**.
2. Insert the batteries.
3. The sensor boots, measures temperature/humidity/voltage, connects to Wi-Fi/MQTT, delivers its reading to Home Assistant in ~1.5 seconds, and enters low-power **Deep Sleep** ($~25\ \mu\text{A}$, completely cool to the touch).
4. The sensor wakes automatically on its internal timer every **10 minutes** to refresh readings in Home Assistant. All cards remain visible and active continuously.

#### 2. Maintenance / Configuration Mode (Stay Awake Mode - Web UI Access):
1. In Home Assistant, toggle the **"Stay Awake"** switch to **`ON`** (the setting is retained on the MQTT broker).
2. **Remove and reinsert one battery** to power-cycle the sensor.
3. When the sensor boots and connects to MQTT, it reads `Stay Awake == ON`, skips deep sleep, and **stays online continuously at `http://192.168.20.20/`** with the full OpenBeken web panel and OTA interface active.
4. When finished with configuration or firmware updates:
   * Toggle **"Stay Awake"** back to **`OFF`** in Home Assistant.
   * Remove and reinsert the battery to place the sensor back into battery-saving Deep Sleep mode.
