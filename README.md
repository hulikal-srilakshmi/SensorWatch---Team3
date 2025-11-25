
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


---

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

## **Estimated Lines of Code**


| **Component**        | **Files** | **Est. Lines** | **Description**                          |
|----------------------|-----------|----------------|------------------------------------------|
| Main Application     | 1         | ~3,200         | Core firmware logic in `src/main.cpp`    |
| Custom Libraries     | 4         | ~1,000         | IMUFX, IMUFX_UI, NeopixelFX, PiezoFX     |
| Test Programs        | 3         | ~300           | `test_bmi160.cpp`, `test_bmi160_daq.cpp`, `temp_real_main.cpp` |
| Web / HTML Assets    | 2         | ~200           | `data/imu.html`, small UI assets         |
| Config / Metadata    | 4         | ~150           | `wifi_config.json`, `platformio.ini`, workspace, misc. |
| **Total (Approx.)**  | **14**    | **~4,800**     | Overall estimated codebase size          |







# *Scan without custom rules*
<img width="1596" height="963" alt="image" src="https://github.com/user-attachments/assets/d57c43a5-747e-46cb-ad67-d3d6f868f9ae" />


<img width="1117" height="956" alt="image" src="https://github.com/user-attachments/assets/4bdd6d84-b61a-4062-9c5b-f5afb0b9d1a0" />


# *Scan with custom rules*

<img width="1636" height="590" alt="image" src="https://github.com/user-attachments/assets/63c7bc94-0e53-465c-9024-fb5c85cc3034" />




# Semgrep Vulnerability Report

This report consolidates findings from:
- `semgrep-results.txt`
- `ota-results.txt`
- `ps-results.txt`
- `web-results.txt`

All vulnerabilities include **severity**, **count**, **CVSS**, **impact**, and **recommendation**.

---

## Vulnerability Summary 

| Vulnerability Name | Severity | Count | CVSS | Impact | Recommendation |
| --- | --- | --- | --- | --- | --- |
| Hardcoded credentials in firmware/logs | CRITICAL | 26 | 9.8 | Credentials embedded in firmware/logs allow takeover of WiFi, APIs, backend. | Remove hardcoded secrets, load from secure storage, rotate exposed keys. |
| Hardcoded WiFi credentials | CRITICAL | 6 | 9.8 | Extraction allows joining same WiFi/AP impersonation → full device/network compromise. | Implement secure provisioning, eliminate defaults, require user-provided creds. |
| Unauthenticated OTA update endpoint | CRITICAL | 2 | 9.8 | Allows arbitrary firmware uploads → persistent malicious firmware. | Strong auth (mTLS/tokens), require cryptographically signed firmware. |
| Hardcoded credentials in telemetry client | CRITICAL | 2 | 9.8 | Hardcoded API keys enable attacker access to backend endpoints. | Remove keys from code, rotate, use per-device secrets. |
| OTA UI route without authentication | CRITICAL | 1 | 9.8 | Browser UI can upload firmware without auth → full takeover. | Enforce strict auth, restrict network exposure. |
| OTA `/update` raw route unauthenticated | CRITICAL | 1 | 9.8 | POSTing firmware without auth gives attackers code execution. | Lock endpoint behind authentication + signature checks. |
| Filesystem admin panel without auth | CRITICAL | 1 | 9.1 | Exposes upload/delete/format → complete storage compromise. | Add strong auth, role checks, compile-out for production. |
| Unencrypted credential storage | HIGH | 5 | 7.5 | WiFi/API keys in cleartext on flash → easy extraction. | Encrypt with device keys, rotate, mask in logs. |
| Unprotected filesystem writes | HIGH | 4 | 7.5 | Attackers can overwrite config/data. | Auth required, validate filenames, log operations. |
| Unprotected filesystem format | HIGH | 2 | 8.2 | Attackers can wipe all device data. | Protect behind admin auth + confirmation guardrails. |
| Weak OTA validation (no size/type/integrity checks) | HIGH | 2 | 8.0 | Malformed/malicious image can brick or hijack device. | Enforce max size, MIME checks, mandatory checksum/signature. |
| Unprotected filesystem delete | HIGH | 2 | 7.5 | Attackers wipe config/logs/data. | Auth + validation + audit logs. |
| Plain HTTP backend | HIGH | 1 | 7.5 | Exposes traffic to MITM/sniffing. | Switch to HTTPS + certificate validation. |
| WebSocket control channel without auth | HIGH | 1 | 8.0 | Real-time remote control exposed without protection. | Require auth tokens, validate origin, restrict commands. |
| OTA update without signature check | HIGH | 1 | 8.6 | Device accepts unsigned firmware. | Implement RSA/ECDSA signing + public key verification. |
| PowerShell Invoke-Expression | HIGH | 1 | 8.0 | Allows arbitrary code execution. | Remove IEX, use direct cmdlets, validate inputs. |
| Plain HTTP URLs in scripts | MEDIUM | 4 | 5.9 | Sensitive data exposed to sniffing/tampering. | Replace with HTTPS + enforce TLS validation. |
| Remote reboot route without access control | MEDIUM | 2 | 5.0 | Attackers can DoS device by repeatedly rebooting it. | Admin-only auth + rate limiting + logging. |
| Inline scripts (XSS surface) | MEDIUM | 2 | 5.4 | Inline JS breaks CSP and allows XSS if data injected. | External JS files + strict CSP. |
| Dynamic JS exec logic | MEDIUM | 1 | 5.3 | Possible injection if user input flows in. | Validate inputs, remove dynamic parsing. |
| PowerShell Start-Process dynamic args | MEDIUM | 1 | 6.5 | Injection via arguments. | Whitelist args, avoid string concatenation. |
| Remote PS execution (Invoke-Command) | MEDIUM | 1 | 6.8 | Lateral movement risk. | Lock hosts, enforce strong auth. |
| Weak hash algorithms (MD5/SHA1) | MEDIUM | 1 | 5.0 | Hash collisions → bypass integrity checks. | Upgrade to SHA-256. |
| Unsafe C-style casts | LOW | 25 | 2.5 | Undefined behavior, possible memory corruption. | Replace with static_cast/reinterpret_cast, validate ranges. |
| Manual memory mgmt without delete | LOW | 11 | 3.1 | Memory leaks → long-term instability / soft DoS. | Use RAII, smart pointers. |
| Hardcoded IP address | LOW | 2 | 3.1 | Magic IP values cause unpredictable behavior. | Replace with proper network state/error handling. |

---

## Executive Summary

- **CRITICAL:** 39 findings  
- **HIGH:** 39 findings  
- **MEDIUM:** 16 findings  
- **LOW:** 38 findings  

The most dangerous issues involve:

1. Hardcoded passwords, WiFi creds, API keys  
2. Unauthenticated OTA & admin routes  
3. Unsigned firmware + no validation  
4. Cleartext storage of sensitive data  
5. Use of insecure PowerShell constructs (IEX, dynamic args)

Immediate remediation should focus on:

- Strong authentication for all management/OTA/admin endpoints  
- Removing and rotating all hardcoded credentials  
- Mandatory cryptographic firmware signing  
- Encrypting credentials at rest  
- Migrating all HTTP → HTTPS  

---

## Detailed Findings Per Severity

### 🔥 CRITICAL Vulnerabilities
(Include detailed explanations from the table above)

### 🚨 HIGH Vulnerabilities
(Include detailed explanations from the table above)

### ⚠️ MEDIUM Vulnerabilities
(Include detailed explanations from the table above)

### 🟢 LOW Vulnerabilities
(Include detailed explanations from the table above)

---

## End of Report


