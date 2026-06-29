# EMG-Muscle-Activity-Monitoring-System
A low-cost, portable EMG rehabilitation monitoring system built with ESP32. It tracks muscle activity, fatigue, repetitions, hold time, and recovery in real time, displays data on an OLED, logs sessions to an SD card, and integrates with Blynk and Streamlit for remote monitoring and visualization.
# 🫀 EMG Rehabilitation Monitoring System

A complete embedded + dashboard system for tracking muscle rehabilitation progress using an ESP32-based EMG sensor device and a Python Streamlit web dashboard.

---

##  Overview

This project consists of two parts:

| Part | Description |
|---|---|
| **ESP32 Firmware** | Reads EMG signals, calibrates, tracks sessions, logs to SD card, syncs time via WiFi, and pushes data to Blynk |
| **Streamlit Dashboard** | Python web app that reads the CSV from the SD card and displays interactive charts, KPIs, and rehabilitation progress analysis |

---

##  System Architecture

```
EMG Sensor (GPIO 34)
        │
        ▼
   ESP32 Microcontroller
        │
        ├──► SH1106 OLED Display       (live session feedback)
        ├──► DS3231 RTC Module          (accurate date & time)
        ├──► Micro SD Card Module       (session CSV logging)
        ├──► WiFi (NTP)                 (auto time sync on boot)
        └──► Blynk App (WiFi)           (remote session summary)
                │
                ▼
         emg_log.csv (SD Card)
                │
                ▼
     Streamlit Dashboard (PC/Browser)
```

---

##  Hardware Required

| Component | Details |
|---|---|
| ESP32 Dev Board | Any standard 38-pin ESP32 |
| EMG Sensor Module | Analog output, connected to GPIO 34 |
| SH1106 OLED Display | 128×64, I2C |
| DS3231 RTC Module | I2C, with CR2032 coin cell battery |
| Micro SD Card Module | SPI interface |
| Micro SD Card | 32GB or less, FAT32 formatted |
| Push Button | Connected to GPIO 27 |

---

##  Wiring

### OLED (SH1106) — I2C
| OLED Pin | ESP32 Pin |
|---|---|
| VCC | 3.3V |
| GND | GND |
| SDA | GPIO 21 |
| SCL | GPIO 22 |

### RTC (DS3231) — I2C
| RTC Pin | ESP32 Pin |
|---|---|
| VCC | 3.3V |
| GND | GND |
| SDA | GPIO 21 |
| SCL | GPIO 22 |

### SD Card Module — SPI
| SD Pin | ESP32 Pin |
|---|---|
| VCC | 3.3V |
| GND | GND |
| SCK | GPIO 18 |
| MISO | GPIO 19 |
| MOSI | GPIO 23 |
| CS | GPIO 5 |

### Button
| Button Pin | ESP32 Pin |
|---|---|
| One side | GPIO 27 |
| Other side | GND |

### EMG Sensor
| EMG Pin | ESP32 Pin |
|---|---|
| Signal OUT | GPIO 34 |
| VCC | 3.3V |
| GND | GND |

---

##  Project Structure

```
emg-rehab-monitor/
│
├── firmware/
│   └── emg_rehab_monitor23_final.ino    # ESP32 Arduino firmware
│
├── dashboard/
│   └── emg_dashboard.py                 # Streamlit web dashboard
│
├── sample_data/
│   └── emg_log.csv                      # Example CSV for testing
│
└── README.md
```

---

##  Firmware Setup

### 1. Arduino Libraries Required

Install these via Arduino IDE → Library Manager:

| Library | Purpose |
|---|---|
| `U8g2` | SH1106 OLED display |
| `RTClib` | DS3231 RTC |
| `BlynkSimpleEsp32` | Blynk IoT integration |
| `SD` | SD card logging |
| `Preferences` | ESP32 flash/NVS storage |
| `WiFi` | WiFi connection |
| `time.h` | NTP time sync (built-in) |

### 2. Configure Your Credentials

Open `emg_rehab_monitor23_final.ino` and update:

```cpp
#define BLYNK_TEMPLATE_ID   "your_template_id"
#define BLYNK_TEMPLATE_NAME "EMG Rehab Monitor"
#define BLYNK_AUTH_TOKEN    "your_auth_token"

const char* WIFI_SSID = "your_wifi_name";
const char* WIFI_PASS = "your_wifi_password";
```

### 3. Upload

- Select **ESP32 Dev Module** as the board
- Select the correct COM port
- Click Upload

---

##  Session Flow

```
Power ON
    │
    ▼
WiFi Connect → NTP Time Sync → RTC Updated
    │
    ▼
[Press Button] → REST CALIBRATION (8 sec — relax muscle)
    │
    ▼
[Press Button] → MVC CALIBRATION (8 sec — max contraction)
    │
    ▼
[Press Button] → MONITORING (live session)
    │
    ▼
[Press Button] → SESSION END (4-page summary on OLED)
    │
    ▼
Data logged to SD card (emg_log.csv)
Data pushed to Blynk app
    │
    ▼
[Press Button] → Back to start for next session
```

---

##  Metrics Tracked Per Session

| Metric | Description |
|---|---|
| **Avg Activation %** | Average muscle activation relative to calibrated MVC |
| **Best MVC** | Peak RMS signal recorded during the session |
| **Strength Score** | Activation mapped to a 1–10 scale |
| **Fatigue %** | Drop in output from session peak (amplitude-based proxy) |
| **Hold Time** | Cumulative seconds above 50% activation |
| **Reps** | Contraction count via hysteresis threshold detection |
| **Duration** | Total session time in seconds |
| **Recovery %** | Current Best MVC ÷ first-ever session Best MVC × 100 |

---

##  Recovery Baseline System

- The **first session ever completed** sets the permanent baseline MVC
- All future sessions compute **Recovery %** against that baseline
- Baseline is stored in **ESP32 flash (NVS)** — survives power loss and SD card swaps
- To reset the baseline: **hold the button for 10 seconds** on the idle screen
- Recovery > 100% means the patient is stronger than when they started

---

##  Automatic Time Sync

Every time the ESP32 powers on and connects to WiFi, it automatically fetches the correct **IST time** from the internet (NTP) and sets it into the DS3231 RTC. No manual time setting needed.

If WiFi is unavailable, the RTC continues from its last known time using the onboard coin cell battery.

---

##  CSV Log Format

Sessions are saved to `/emg_log.csv` on the SD card:

```
Date,Time,AvgActivation,BestMVC,StrengthScore,Fatigue,HoldTime,Reps,Duration,Recovery
01/01/2025,09:15:00,42,85,4,18,12.5,8,180,100
05/01/2025,10:00:00,45,88,5,20,14.0,9,195,103
```

---

##  Blynk App Setup

| Virtual Pin | Data |
|---|---|
| V0 | Avg Activation % |
| V1 | Best MVC |
| V2 | Rep Count |
| V3 | Fatigue % |
| V4 | Hold Time (s) |
| V5 | Duration (s) |
| V6 | Session Date |

---

##  Streamlit Dashboard Setup

### 1. Install dependencies

```bash
pip install streamlit plotly pandas
```

### 2. Run the dashboard

```bash
cd path/to/dashboard
streamlit run emg_dashboard.py
```

### 3. Open in browser

```
http://localhost:8501
```

### 4. Upload your CSV

Click **Browse files** on the landing page and select `emg_log.csv` from your SD card.

---

##  Dashboard Features

- **Upload landing page** — clean upload screen before dashboard loads
- **8 KPI cards** — latest session metrics at a glance
- **Patient status badge** — Weak Recovery / Improving / Good Progress / Excellent Recovery
- **Recovery progress chart** — line chart with 100% baseline reference
- **MVC, Fatigue, Activation, Duration** — line charts per session
- **Reps and Hold Time** — bar charts per session
- **Progress analysis** — auto-generated ▲/▼ comparison from first to latest session
- **Session history table** — all sessions in a sortable, scrollable table

---

##  Patient Status Thresholds

| Recovery % | Status |
|---|---|
| < 90% | 🔴 Weak Recovery |
| 90% – 120% | 🟡 Improving |
| 120% – 150% | 🟢 Good Progress |
| > 150% | 🟣 Excellent Recovery |

---

##  Troubleshooting

| Problem | Fix |
|---|---|
| SD Card FAIL on OLED | Insert FAT32 formatted micro SD card and restart |
| Wrong time/date | Check WiFi connection — NTP syncs on every boot |
| RTC shows random time | Replace CR2032 coin cell battery |
| MVC TOO LOW error | Contract muscle harder during MVC calibration |
| Blynk OFFLINE | Check WiFi credentials and Blynk auth token |
| Dashboard file not found | Run `cd` to the correct folder before `streamlit run` |

---

## 🙌 Acknowledgements

- [U8g2 Library](https://github.com/olikraus/u8g2) — OLED display
- [RTClib](https://github.com/adafruit/RTClib) — DS3231 RTC
- [Blynk](https://blynk.io) — IoT dashboard
- [Streamlit](https://streamlit.io) — Web dashboard framework
- [Plotly](https://plotly.com) — Interactive charts
