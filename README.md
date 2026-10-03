# Tuya Temperature & Humidity Sensor (BK7231N / SHT30) - OpenBeken Guide

Comprehensive documentation for flashing, pinout configuration, Home Assistant MQTT integration, battery chemistry calibration, and deep-sleep optimization for the **Tuya Generic Temperature and Humidity Sensor (v1.1.17)** based on the **Beken BK7231N (CBU)** microcontroller.

---

## 1. Hardware Specifications

| Component | Detail |
| :--- | :--- |
| **Microcontroller** | Beken BK7231N (Tuya CBU Module) |
| **Flash Memory** | 2048 KiB (2 MB) |
| **Sensor IC** | Sensirion SHT30 / SHT3x (I2C) |
| **Power Supply** | 2x AAA Batteries (Alkaline: 2.0–3.0V, or NiMH Rechargeable: 2.0–2.5V) |
| **Status LED** | Red LED on **P26** (Active-High: 0V = OFF, 3.3V = ON) |
| **Pair / Wake Button**| Momentary Tactile Switch on **P20** (Verified active-low GPIO via `P20_HOLD`) |
| **Sensor Power Switch**| Transistor switch on **P17** (Active-High: powers ADC divider network) |
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

The hardware pinout verified on the physical hardware:
| Pin | OpenBeken Role | Role ID | Channels | Description |
| :--- | :--- | :--- | :--- | :--- |
| **P7** | `SHT3X_SCK` | 49 | — | I2C Clock |
| **P8** | `SHT3X_SDA` | 48 | **Ch 1, Ch 2** | I2C Data (Channel 1 = Temp, Channel 2 = Humidity) |
| **P14**| `DoorSnsrWSleep` | 58 | **Ch 0** | Optional wake input on PCB rev B |
| **P17**| `BAT_Relay` | 51 | — | Battery ADC Resistor Divider Switch (Active-High, isolates divider to prevent parasitic sleep drain) |
| **P20**| `DoorSnsrWSleep` | 58 | **Ch 0** | Physical Reset / Pair Button (Verified active-low GPIO via `P20_HOLD`) |
| **P23**| `BAT_ADC` | 60 | — | Battery Voltage ADC (ADC3, monitored by OpenBeken Battery driver) |
| **P26**| `AlwaysLow` | 35 | — | Red Status LED (0V = OFF, prevents parasitic drain) |
| *All others* | `None` | 0 | — | Unassigned / High Impedance |

### Channel Types
* **Channel 1**: `Temperature_div10` (interprets raw `267` as `26.7 °C`)
* **Channel 2**: `Humidity` (interprets raw `36` as `36 %`)
*(Note: Channel 3 is not used for battery/voltage because the OpenBeken `Battery` driver manages ADC sampling, divider scaling, and MQTT publishing directly to `voltage` and `battery` topics).*

### OpenBeken CLI Configuration Commands
```bash
# Pin roles and channels
SetPinRole 7 SHT3X_SCK
SetPinRole 8 SHT3X_SDA
SetPinChannel 8 1 2
SetPinRole 14 DoorSnsrWSleep
SetPinRole 17 BAT_Relay
SetPinRole 20 DoorSnsrWSleep
SetPinRole 23 BAT_ADC
SetPinRole 26 AlwaysLow

# Clean up unused pins
SetPinRole 16 None
SetPinRole 22 None

# Channel formatting
SetChannelType 1 Temperature_div10
SetChannelType 2 Humidity

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
| `sensor.obk55a19113_voltage` | Voltage | Sensor | Battery Pack Voltage in `mV` |
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

## 5. Battery Chemistry Calibration: NiMH vs. Alkaline

The Beken BK7231N ADC measures battery voltage through a 1:1 resistor voltage divider on **Pin 23 (ADC3)**. The voltage is configured using `Battery_Setup`:

```text
Battery_Setup [minbatt] [maxbatt] [V_divider] [Vref] [AD Bits]
```

* **`minbatt`**: Minimum operational voltage in mV (0% battery cut-off).
* **`maxbatt`**: Maximum operational voltage in mV (100% full charge).
* **`V_divider`**: Resistor voltage divider ratio (`2.0` for 1:1 equal resistors).
* **`Vref`**: ADC reference voltage (`2400` mV).
* **`AD Bits`**: ADC resolution (`4096` for 12-bit ADC).

### Chemistry Profiles:

1. **Rechargeable NiMH Batteries (2x AAA in Series)**:
   * **Nominal Voltage**: $1.2\text{V}$ per cell ($2.4\text{V}$ total).
   * **Full Charge**: $\approx 1.25\text{V} - 1.30\text{V}$ per cell (**$2500\text{ mV}$** total).
   * **Depleted / Cutoff**: $\approx 1.0\text{V}$ per cell (**$2000\text{ mV}$** total).
   * **Command**:
     ```text
     Battery_Setup 2000 2500 2.0 2400 4096
     ```
   * *Why fresh NiMH batteries previously reported 40–50%*: When calibrated for 3.0V alkaline, a freshly charged 2.45V NiMH pack evaluates to $(2450 - 2000) / (3000 - 2000) = 45\%$. With the 2500 mV ceiling, it correctly reports **$90\text{–}95\%$**.

2. **Standard Alkaline Batteries (2x AAA in Series)**:
   * **Full Charge**: $1.5\text{V} - 1.6\text{V}$ per cell (**$3000\text{ mV}$** total).
   * **Depleted / Cutoff**: $1.0\text{V}$ per cell (**$2000\text{ mV}$** total).
   * **Command**:
     ```text
     Battery_Setup 2000 3000 2.0 2400 4096
     ```

---

## 6. Power Consumption, Thermals, and Deep Sleep

### The Always-On vs Deep Sleep Physics
* In **Always-On** mode, the Wi-Fi transceiver and CPU run 24/7, drawing **80–100 mA** continuous current ($~0.25\text{ W}$). On 2× AAA batteries (1000 mAh), the batteries are completely drained in **12–24 hours**.
* In **Deep Sleep**, the SoC disables its radio, CPU, and clocks, dropping power consumption to **$\approx 20\text{–}30\ \mu\text{A}$**.
* **Why Wi-Fi Sensors Drain Quickly Without Hardening**:
  1. **The Infinite Wait Trap**: `waitFor WiFiState 4` and `waitFor MQTTState 1` have no timeout. If a Wi-Fi router channel hops or drops a packet, the sensor spins forever drawing 120 mA until the batteries die.
  2. **Active Time Overhead**: Waking for 4.5 seconds every 10 minutes burns substantial energy over 144 daily cycles.

### Hardened Production `autoexec.bat` (30-Minute Cadence + 6s Watchdog)
Set this script in OpenBeken (**Config $\rightarrow$ Change Startup Command Text**):

```batch
; Enable low power 802.11 modem sleep
PowerSave 1

; Optimize MQTT and WiFi quick connect
SetFlag 35 1    ; Omit MQTT availability topic (values persist in HA cards)
SetFlag 7 1     ; Quick connect
SetFlag 37 1    ; Fast connect (caches BSSID and RF channel in flash to skip 13-channel scan)

; Start drivers
startDriver SHT3X
startDriver Battery
; Calibrated for 2x AAA NiMH rechargeable batteries (2000mV to 2500mV):
Battery_Setup 2000 2500 2.0 2400 4096

; Link physical button to Channel 5 (Stay Awake switch) - maps both P14 and P20
addEventHandler OnClick 14 "toggleChannel 5"
addEventHandler OnHold 14 "toggleChannel 5"
addEventHandler OnClick 20 "toggleChannel 5"
addEventHandler OnHold 20 "toggleChannel 5"
DSEdge 1 14
DSEdge 1 20

; Hard fallback watchdog: If not finished within 6 seconds, abort and sleep immediately!
; (Guarantees the sensor can NEVER get stuck awake draining batteries on network hiccup)
addRepeatingEvent 6 1 if $CH5==0 then PinDeepSleep 1800

; Wait for network connection
waitFor WiFiState 4
waitFor MQTTState 1

; Capture fresh sensor readings
SHT_Measure
delay_ms 250

; Publish channel data and named sensor topics to MQTT
publishChannels
publishFloat "temperature" $CH1
publishFloat "humidity" $CH2

; Short 400ms buffer flush (replaces the old 2000ms delay)
delay_ms 400

; Sleep for 30 minutes (1800s) if Stay Awake is OFF
if $CH5==0 then PinDeepSleep 1800
```

### Summary of Efficiency Optimizations:
1. **Hard 6-Second Watchdog**: `addRepeatingEvent 6 1 if $CH5==0 then PinDeepSleep 1800` protects against network hangs. If Wi-Fi or MQTT takes more than 6 seconds, the device immediately aborts and returns to sleep.
2. **Reduced Awake Duration**: Lowered from ~4.5 seconds to **~1.8–2.0 seconds** per wake cycle.
3. **Reduced Wake Frequency**: Waking every 30 minutes (48 times/day) rather than every 10 minutes (144 times/day) yields an instant **$3\times$ energy reduction**.
4. **Isolated Resistor Divider**: Pin 17 (`BAT_Relay`) powers the voltage divider only for 10 ms during ADC sampling, eliminating parasitic drain during sleep.
5. **Expected Battery Life**: Extends 2× AAA NiMH battery life from **12–24 hours to 4–6+ months**.

---

## 7. Operating Procedure (How to Switch Modes & Wake Up)

Because the sensor is in low-power deep sleep for 99.8% of the time, mode switching is controlled via the Home Assistant switch and the physical switch:

#### 1. Waking the Sensor from Deep Sleep:
* **The 1-Second Hold Rule**: The BK7231N deep sleep wake interrupt is routed to **Pin 20** through an internal low-power analog comparator filter. Quick micro-taps (< 150 ms) are rejected as electrical noise. **Press and hold the button for 1 to 2 seconds** to reliably wake the device.
* **The Rail Capacitor Discharge Rule**: When replacing or reseating batteries, the board's decoupling capacitors take **30 to 45 seconds** to discharge at 25 µA. Pulling the battery for only 2 seconds causes brownout latchup rather than a clean Power-On Reset. **To cold-boot instantly: remove one battery, hold down the reset button for 5 seconds to drain the rail, then reinsert the battery.**

#### 2. Normal Battery Operation (Deep Sleep Mode - 4 to 6+ Months Battery Life):
1. In Home Assistant, ensure the **"Stay Awake"** switch is set to **`OFF`**.
2. The sensor sleeps in low power ($~25\ \mu\text{A}$, completely cool to the touch).
3. The sensor wakes automatically on its internal timer every **30 minutes** (or immediately when the button on Pin 20 is held for 1–2s) to refresh readings in Home Assistant. All dashboard cards remain visible continuously.

#### 3. Maintenance / Configuration Mode (Stay Awake Mode - Web UI Access):
1. In Home Assistant, toggle the **"Stay Awake"** switch to **`ON`** (the setting is retained on the MQTT broker).
2. **Remove one battery, hold the button for 5s to discharge, and reinsert** (or hold down the physical button for 3–5 seconds during boot).
3. When the sensor boots and connects to MQTT, it reads `Stay Awake == ON`, skips deep sleep, and **stays online continuously at `http://192.168.20.20/`** with the full OpenBeken web panel and OTA interface active.
4. When finished with configuration or firmware updates:
   * Toggle **"Stay Awake"** back to **`OFF`** in Home Assistant.
   * Remove and reinsert the battery (with capacitor discharge) to place the sensor back into battery-saving Deep Sleep mode.
