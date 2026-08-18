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
| **Pair / Wake Button**| Momentary Tactile Switch on **P20** (Active-Low) |
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
| **P20**| `Btn_n` | 4 | **Ch 0** | Pairing / Wake / Safe-Mode Button (Active-Low) |
| **P23**| `BAT_ADC` | 60 | **Ch 3** | Battery Voltage ADC (linked to Channel 3) |
| **P26**| `AlwaysLow` | 35 | — | Red Status LED (0V = OFF, prevents parasitic drain) |
| *All others* | ` ` (None) | 0 | — | Unassigned / High Impedance |

### Channel Types
* **Channel 1**: `Temperature_div10` (interprets raw `267` as `26.7 °C`)
* **Channel 2**: `Humidity` (interprets raw `36` as `36 %`)
* **Channel 3**: `Voltage` (Battery ADC voltage)

### OpenBeken CLI Configuration Commands
```bash
# Pin roles and channels
SetPinRole 7 SHT3X_SCK
SetPinRole 8 SHT3X_SDA
SetPinChannel 8 1 2
SetPinRole 17 AlwaysHigh
SetPinRole 20 Btn_n
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

| Entity ID | Name | Measurement | Unit |
| :--- | :--- | :--- | :--- |
| `sensor.obk55a19113_temperature` | Temperature | Ambient Temperature (SHT30) | `°C` |
| `sensor.obk55a19113_humidity` | Humidity | Ambient Humidity (SHT30) | `%` |
| `sensor.obk55a19113_voltage` | Voltage | Battery ADC Voltage | `mV` |
| `sensor.obk55a19113_battery` | Battery | Battery Level | `%` |
| `sensor.obk55a19113_temperature_3`| Temperature (Diag) | Internal SoC Die Temperature | `°C` |
| `sensor.obk55a19113_rssi` | RSSI | Wi-Fi Signal Strength | `dBm` |
| `sensor.obk55a19113_uptime` | Uptime | System Uptime | `s` |
| `sensor.obk55a19113_ip` | IP | Device IP Address | — |

---

## 5. Power Consumption, Thermals, and Deep Sleep

### The Always-On Problem (Why the chip feels warm)
* In **Always-On** mode, the Wi-Fi transceiver and CPU run 24/7, drawing **80–100 mA** continuous current ($~0.3\text{ W}$).
* Inside the compact enclosed sensor casing, this heats the silicon die to **$\approx 45^\circ\text{C}$** (visible under `sensor.obk55a19113_temperature_3`) and exhausts standard AAA batteries in **12 to 15 hours**.

### Deep Sleep Solution (6–12+ Months Battery Life)
In **Deep Sleep**, the SoC disables its radio and clocks, dropping power consumption to **$\approx 20\text{–}30\ \mu\text{A}$**. The device remains completely cold (room temperature) and wakes only for 1.5–2 seconds per cycle to transmit data.

### Production `autoexec.bat` / Startup Script
Set this script in OpenBeken (**Config $\rightarrow$ Change Startup Command Text**):

```batch
; Enable low power 802.11 modem sleep
PowerSave 1

; Start drivers
startDriver SHT3X
startDriver Battery
Battery_Setup 2000 3000 2.29 2400 4096

; Wait for Wi-Fi and MQTT connection
waitFor WiFiState 4
waitFor MQTTState 1

; Capture sensor reading and publish to Home Assistant
SHT_Measure
publishChannels

; Enter Deep Sleep for 10 minutes (600 seconds)
DeepSleep 600
```

### Safe Mode Recovery (Accessing Web UI after Deep Sleep is Enabled)
Once `DeepSleep` is active, the web server is offline while sleeping. If you need to access the web UI at `http://192.168.20.20/` again:
1. Remove one AAA battery.
2. **Press and hold the physical button (P20)**.
3. Reinsert the battery while keeping the button held for 3–5 seconds.
4. OpenBeken will boot into **Safe Mode**, keeping the web server active and disabling the sleep timer so you can reconfigure or update firmware.
# Openbeken_temp_humidity_sensor
