
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





# **Data Flow Diagram**

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
─────────────────┼────────────────────────
                 |
                 v
            HARDWARE LAYER
    DS18B20 | BMI160 | NeoPixel | Piezo



# **System Architecture**


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
│  ⇣  Sends Commands / Receives Live Data (HTTP + WebSocket over WiFi)      │
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
│                                                                           │
│  ⇣ Data & Signals via GPIO / I²C / SPI / 1-Wire                            │
└───────────────────────────────────────────────────────────────────────────┘

                           OVERALL DATA FLOW
           ┌────────────────────────────────────────────────┐
           │ User Browser ⇄ ESP32 Firmware ⇄ Hardware Layer │
           └────────────────────────────────────────────────┘




