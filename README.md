
# **SensorWatch Security Audit Report**


**Date:** November 2025   
**Auditor:** Srilakshmi Hulikal  


**Repository:** https://github.com/WinWinLabs/SensorWatch/tree/team-3-staging  
**Platform:** ESP32 / ESP32-S3  
**Framework:** PlatformIO + Arduino Core  

# **Executive Summary** 

SensorWatch Team 3 – Staging is an ESP32-based IoT firmware that integrates temperature sensing, IMU motion data, NeoPixel LED control, WiFi configuration, and a full web-based management interface. The system relies on a large monolithic `main.cpp` file, multiple custom device-control libraries, and several HTML/JavaScript UIs stored both in LittleFS and within the firmware.

The current architecture provides rich functionality but also exposes a broad attack surface. Several critical risks arise from missing authentication, permissive filesystem access, lack of encryption for WiFi credentials, insecure configuration flows, and unvalidated client inputs across WebSocket and HTTP endpoints.

This Executive Summary highlights the most important findings:
- The device exposes sensitive operations (filesystem, connectivity, configuration) without authentication.
- WiFi credentials are stored in plaintext and transmitted without encryption.
- WebSocket and HTTP endpoints allow user-controlled data without sanitization.
- File uploads and deletes are unrestricted, enabling overwrite or deletion of core system files.
- Large embedded HTML/JS interfaces increase exposure to XSS and client-side injection.
- No rate limiting or access control mechanisms are present, enabling trivial DoS conditions.

Overall security posture: **High-Risk / Requires Immediate Hardening** 



# **System Architecture**


```text
┌───────────────────────────────────────────────────────────────────────────┐
│                               USER INTERFACE                              │
│───────────────────────────────────────────────────────────────────────────│
│  Web Browser (Desktop/Mobile)                                             │
│   • Dashboard & Charts                                                    │
│   • IMU 3D Motion View                                                    │
│   • LED / Neopixel Control Panel                                          │
│   • WiFi Settings & Device Config                                         │
│   • File Manager (Upload/Delete/Format)                                   │
│                                                                           │
│  ⇣ Sends Commands / Receives Live Data (HTTP + WebSocket over WiFi)       │
└───────────────────────────────────────────────────────────────────────────┘

                     ╲╱

       ┌──────────────────────────────────────┐
       │      ESP32 FIRMWARE (main.cpp)       │
       │──────────────────────────────────────│
       │  CORE SERVICES                       │
       │   • HTTP Server (AsyncWebServer)     │
       │   • WebSocket Server (JSON streams)  │
       │   • WiFi Manager (AP/STA switching)  │
       │   • File System (LittleFS)           │
       │   • Sensor Capture Loop              │
       │                                      │
       │  FEATURE MODULES                     │
       │   • IMUFX / IMUFX_UI   → motion data │
       │   • NeopixelFX        → LED effects  │
       │   • PiezoFX           → buzzer tones │
       │                                      │
       │  SECURITY-RELEVANT AREAS             │
       │   • WiFi config handling             │
       │   • File upload/delete endpoints     │
       │   • Live WebSocket control surface   │
       │   • Stored JSON configs              │
       └──────────────────────────────────────┘

                     ╲╱

┌───────────────────────────────────────────────────────────────────────────┐
│                               HARDWARE LAYER                              │
│───────────────────────────────────────────────────────────────────────────│
│  DS18B20  – Temperature Sensors                                            │
│  BMI160   – IMU (Gyro + Accelerometer)                                     │
│  WS2812B  – NeoPixel RGB LEDs                                              │
│  Piezo    – Buzzer                                                         │
└───────────────────────────────────────────────────────────────────────────┘

```





## **Data Flow Diagram**



```text
                 USER BROWSER
         (Desktop / Mobile, JS UI)
        ----------------------------
        - Dashboards / Charts
        - IMU & LED controls
        - WiFi & config pages
        - File manager UI
                 |
                 |  HTTP + WebSocket
                 |  (JSON, form data)
─────────────────┼────────────────────────
   TRUST BOUNDARY #1 – Untrusted Client
─────────────────┼────────────────────────
                 |
                 v
            ESP32 FIRMWARE
           (main.cpp core)
     ------------------------------
     - HTTP server / WebSocket
     - WiFi manager
     - LittleFS storage
     - Sensor capture loop
     - IMUFX / NeopixelFX / PiezoFX
                 |
                 |  GPIO / I²C / SPI / 1-Wire
─────────────────┼────────────────────────
  TRUST BOUNDARY #2 – Firmware vs Hardware
───────────────────────────────────────────
                 |
                 v
            HARDWARE LAYER
    DS18B20 | BMI160 | NeoPixel | Piezo


```



## **Technology Stack**

```text 
| **Layer**      | **Technology**              | **Purpose** |
|----------------|-----------------------------|-------------|
| **Hardware**   | ESP32-S3                    | Main microcontroller (WiFi, WebSocket, HTTP, GPIO) |
|                | DS18B20                     | Temperature sensors (1-Wire) |
|                | BMI160                      | 6-axis IMU (Gyro + Accelerometer via SPI/I²C) |
|                | WS2812B NeoPixel            | RGB LED output |
|                | Piezo Buzzer                | Audio feedback / alert tones |
| **Firmware**   | Arduino Core (ESP32)        | Development framework |
|                | PlatformIO                  | Build system & dependency manager |
|                | AsyncWebServer              | Non-blocking HTTP server |
|                | AsyncTCP                    | TCP backend for AsyncWebServer |
|                | ArduinoJson                 | JSON parsing / encoding for API & WebSocket |
|                | LittleFS                    | On-chip filesystem (HTML, CSS, JS, JSON) |
|                | OneWire / DallasTemperature | DS18B20 temperature drivers |
| **Libraries**  | IMUFX                       | IMU sensor processing |
|                | IMUFX_UI                    | IMU web interface logic |
|                | NeopixelFX                  | LED animation & effects |
|                | PiezoFX                     | Buzzer tone engine |
| **Web**        | HTML / CSS / JavaScript     | Browser UI hosted from ESP32 |
|                | WebSockets                  | Real-time IMU + sensor streaming |
|                | JSON (HTTP/WebSocket)       | Live telemetry & configuration messages |
| **External API** | HTTP POST (optional)      | Remote server telemetry upload |

```



## **File Structure Overview**


```text
SensorWatch/
├── .github/
│   └── workflows/
│       └── ci-platformio.yml
│
├── .vscode/
│   └── extensions.json
│
├── data/
│   ├── wifi_config.json
│   ├── imu.html
│   └── favicon.ico
│
├── docs/
│
├── include/
│
├── lib/
│   ├── IMUFX/
│   │   ├── IMUFX.cpp
│   │   ├── IMUFX.h
│   │   └── README.md
│   │
│   ├── IMUFX_UI/
│   │   ├── src/
│   │   │   ├── IMUFX_UI.cpp
│   │   │   └── IMUFX_UI.h
│   │   ├── assets/
│   │   └── library.json
│   │
│   ├── NeopixelFX/
│   │   ├── NeopixelFX.cpp
│   │   ├── NeopixelFX.h
│   │   └── library.json
│   │
│   ├── PiezoFX/
│   │   ├── src/
│   │   │   ├── PiezoFX.cpp
│   │   │   └── PiezoFX.h
│   │   └── library.json
│
├── scripts/
│   └── fix_com.p1.ps1
│
├── src/
│   ├── main.cpp
│   ├── SensorWatch.code-workspace
│   ├── test_bmi160_daq.cpp
│   ├── test_bmi160.cpp
│   └── temp_real_main.cpp
│
├── test/
│
├── tools/
│
├── .gitignore
├── platformio.ini
└── CONTRIBUTING.md

```




## **Network Communication Flow**

```text
                  USER BROWSER
           (Desktop / Mobile, JS UI)
          ----------------------------
          - Opens SensorWatch URL
          - Loads HTML/CSS/JS
          - Sends config / control actions
          - Receives live sensor + IMU data
                     |
                     |  HTTP (GET / POST) + WebSocket
                     |  over WiFi
─────────────────────┼──────────────────────────────
  TRUST BOUNDARY #1 – Untrusted Client / Local LAN
─────────────────────┼──────────────────────────────
                     |
                     v
             LOCAL WIFI NETWORK
          (Home / Lab Router or AP)
          ----------------------------
          - Routes HTTP/WebSocket traffic
          - Forwards packets to ESP32
          - May bridge to wider Internet
                     |
                     |  WiFi STA (ESP32 joins LAN)
                     |  or Soft-AP (client joins ESP32)
─────────────────────┼──────────────────────────────
  TRUST BOUNDARY #2 – Network vs Device Firmware
─────────────────────┼──────────────────────────────
                     |
                     v
                 ESP32 DEVICE
               (SensorWatch-3)
          ----------------------------
          - AsyncWebServer (HTTP)
          - WebSocket server (/ws)
          - LittleFS file serving
          - WiFi config endpoints
          - File manager endpoints
          - Optional OTA/update handler
                     |
     ┌───────────────┼─────────────────────┐
     |               |                     |
     v               v                     v
  STATIC UI      CONTROL API         LIVE DATA STREAM
(HTML/CSS/JS)   (HTTP endpoints)     (WebSocket JSON)
  ---------      ---------------     ----------------
- Served from    - /connectivity     - Sensor readings
  LittleFS         (WiFi config)     - IMU data
- Loaded once    - /list-files       - LED / IMU status
  in browser     - /download         - Status updates
                 - /upload-file
                 - /format-fs

                     |
                     |  (optional, if enabled)
                     v
          REMOTE / CLOUD ENDPOINTS
          ----------------------------
          - HTTP POST telemetry
          - Remote dashboards / APIs
```
