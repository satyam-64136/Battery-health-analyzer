# 🔋 Smart AA Battery Analyzer — v8.0

A WiFi-enabled AA battery analyzer built on the **NodeMCU ESP8266**. Insert an AA battery, connect to the device's hotspot, and instantly get open-circuit voltage, loaded voltage, voltage sag, internal resistance, and remaining capacity — all displayed on a live web dashboard and a local I2C LCD.

---

## ✨ Features

- **Open-circuit & loaded voltage** measurement with a MOSFET-switched resistive load
- **Internal resistance (Rᵢ)** estimation via voltage sag under known load
- **Realistic battery percentage** using a non-linear AA alkaline discharge curve
- **Status classification** — `GOOD` / `WEAK` / `REPLACE` based on voltage + Rᵢ
- **Live web dashboard** served directly from the ESP8266 (no internet needed)
- **Captive portal** — dashboard opens automatically on Android, iOS, Windows, and macOS
- **16×2 I2C LCD** with graphical charge bar and flicker-free updates
- **Production-grade signal processing** — 20-sample median-filtered ADC, MOSFET settle delay, Rᵢ rolling average, NaN/overvoltage guards

---

## 🛠️ Hardware

| Component | Detail |
|---|---|
| Microcontroller | NodeMCU ESP8266 |
| Display | 16×2 I2C LCD (address `0x27`, fallback `0x3F`) |
| Load switch | IRFZ44N MOSFET via 220 Ω gate resistor |
| Load resistor | 3.3 Ω (defines load current for Rᵢ measurement) |
| Voltage divider | On **A0** — calibrated for 0–1.5 V AA range (multiplier `6.67`) |

### Pin Assignments

| NodeMCU Pin | Function |
|---|---|
| `A0` | Voltage divider midpoint (ADC input) |
| `D5` | MOSFET gate (load switch) |
| `D1` (SCL) | LCD I2C clock |
| `D2` (SDA) | LCD I2C data |

> ⚠️ **Do not change pin assignments or calibration constants** without re-deriving the `MULTIPLIER` from a known reference voltage.

---

## 📶 Connecting to the Dashboard

1. Power the NodeMCU.
2. On your phone or laptop, connect to the open WiFi network: **`BatteryAnalyser`** (no password).
3. A captive portal will redirect you automatically. If it doesn't, open **`http://192.168.4.1`** in your browser.
4. Insert an AA battery and watch the readings update in real time.

---

## 📊 Web Dashboard

The dashboard polls `/data` every 500 ms and displays:

| Metric | Description |
|---|---|
| **Voltage (OC)** | Open-circuit terminal voltage |
| **Loaded Voltage** | Voltage under 3.3 Ω resistive load |
| **Voltage Sag** | OC − Loaded (indicator of internal resistance) |
| **Internal Resistance** | Rᵢ in Ω, filtered rolling average |
| **Capacity %** | Derived from non-linear AA discharge curve |
| **Health badge** | Excellent / Good / Low / Critical / Dead |
| **Cell state** | New / Good / Aged / Weak (from Rᵢ) |
| **Voltage sparkline** | Live history of OC voltage over time |

---

## 🔌 Firmware

### Dependencies

Install via the Arduino Library Manager or PlatformIO:

| Library | Purpose |
|---|---|
| `ESP8266WiFi` | Built-in with ESP8266 Arduino core |
| `ESP8266WebServer` | Built-in with ESP8266 Arduino core |
| `DNSServer` | Built-in — captive portal DNS |
| `Wire` | Built-in — I2C |
| `LiquidCrystal_I2C` | Frank de Brabander / johnrickman |

### Building & Uploading

1. Install the **ESP8266 Arduino core** via Boards Manager (`http://arduino.esp8266.com/stable/package_esp8266com_index.json`).
2. Select board: **NodeMCU 1.0 (ESP-12E Module)**.
3. Open `BatteryAnalyzer_v8.ino` and upload at **115200 baud**.
4. Open Serial Monitor at 115200 to see boot and measurement logs.

---

## ⚙️ Configuration

All tunable constants are grouped at the top of the sketch:

```cpp
// ADC sampling
#define ADC_SAMPLES      20     // Samples per reading
#define ADC_SAMPLE_DELAY 3      // ms between samples

// Timing
#define MOSFET_SETTLE_MS 150    // Settle time after load ON
#define OC_SETTLE_MS     800    // Recovery time after load OFF

// Voltage thresholds (AA alkaline)
const float BATT_FULL    = 1.55;
const float BATT_DEAD    = 0.90;
const float NO_BATT_THR  = 0.25;
const float OVERVOLT_THR = 2.00;

// Internal resistance filter
#define RINT_MIN         0.0
#define RINT_MAX         20.0
#define RINT_AVG_COUNT   4
```

> 💡 If your LCD stays blank, change the I2C address in `LiquidCrystal_I2C lcd(0x27, 16, 2)` from `0x27` to `0x3F`.

---

## 📋 Changelog

### v8.0
1. **ADC stability** — 20-sample median filter with inter-sample delay (replaces simple average)
2. **Load settle** — 150 ms MOSFET settle before reading (empirically optimised)
3. **Battery %** — Non-linear AA discharge curve (lookup table interpolation)
4. **Rᵢ filtering** — Clamp to `[0, 20] Ω` + 4-reading rolling average
5. **Status field** — `GOOD` / `WEAK` / `REPLACE` added to JSON and classification logic
6. **Fail-safes** — Overvoltage detection, NaN guards on all float fields, anomalous load reading fallback
7. **WiFi** — Open AP `BatteryAnalyser` (no password)
8. **Captive portal** — DNS wildcard redirect + OS-specific probe URL handlers (Android, iOS, Windows, macOS, Kindle)
9. **Web server** — Non-blocking `/data` endpoint; zero ADC/MOSFET activity in HTTP handler
10. **Code quality** — Modular functions, production-safe guards, full Serial debug output

---

## 📐 How It Works

```
Battery inserted
       │
       ▼
MOSFET OFF → wait 800 ms → read V_OC (20-sample median)
       │
       ▼
MOSFET ON → wait 150 ms → read V_loaded (20-sample median)
       │
       ▼
MOSFET OFF immediately

Sag   = V_OC − V_loaded
I     = V_loaded / 3.3 Ω
Rᵢ    = Sag / I  (clamped + rolling average)
%     = non-linear discharge curve lookup
Status = GOOD / WEAK / REPLACE
       │
       ▼
Update LCD + latest{} struct
Web /data endpoint returns latest{} as JSON
```

---

## 📄 License

MIT — free to use, modify, and distribute with attribution.
