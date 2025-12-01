# SensorWatch – Static Analysis Security Report
### Generated From Semgrep Results ONLY  
### Format: Industry-Standard SAST Report (CWE / CVSS / Severity / Evidence / Remediation)

This report contains ONLY findings that appear in:
- semgrep-results.txt  
- ota-results.txt  
- web-results.txt  
- ps-results.txt  

No assumptions, no added vulnerabilities, no extra code review.

---

# 1. Executive Summary

The Semgrep analysis identified multiple classes of security concerns across the firmware codebase.  
These include:

- Hardcoded credentials  
- Hardcoded WiFi configuration  
- Hardcoded plaintext HTTP URLs  
- Unencrypted local storage patterns  
- OTA update endpoints and operations  
- Filesystem routes (fs, delete, format, write)  
- Inline script blocks in HTML  
- WebSocket endpoints  
- PowerShell rule matches (weak hash, Invoke-Expression, Start-Process)

These findings require review but do **NOT** confirm exploitability unless manually validated.

---

# 2. Scope

This report includes only the findings from Semgrep rule family **SensorWatch.\***.

### Files referenced by Semgrep:
- SensorWatch/src/main.cpp  
- SensorWatch/data/imu.html  
- SensorWatch/lib/IMUFX_UI/assets/imu.html  
- PowerShell patterns (from ps-results.txt)  

---

# 3. Methodology

Semgrep was executed against the codebase with a custom ruleset.  
Each Semgrep match is translated here into a standardized SAST report entry:

- Severity (Industry baseline)  
- CWE Mapping  
- CVSS Score (Base v3.1 approximation)  
- Impact Description  
- Exact Evidence (taken from Semgrep output)  
- Recommended Remediation  

This document contains:
- CRITICAL Findings  
- HIGH Findings  
- MEDIUM Findings  
- LOW Findings  

All evidence comes directly from:
- semgrep-results.txt  
- ota-results.txt  
- web-results.txt  
- ps-results.txt  

No assumptions. No manual vulnerabilities added.

---

## CRITICAL SEVERITY FINDINGS

## Category: OTA Security

---

## 1. OTA Update Endpoint  

**Severity:** CRITICAL  
**CWE:** CWE-306 – Missing Authentication  
**CVSS:** 9.8  

**Location:** `SensorWatch/src/main.cpp`

**Evidence:**
```text
1972 server.on("/update", HTTP_POST,
1994 if (!Update.begin(UPDATE_SIZE_UNKNOWN)) {
```

**Impact:** 
Firmware flashing occurs without cryptographic integrity checks. Attackers can upload malicious firmware that fully compromises system integrity.

**Recommendation:**  
Implement firmware signature verification (RSA/ECDSA) and reject any unauthenticated or tampered images.



## 2. OTA Update Logic (Begin / Write / End)


**Severity:** CRITICAL  
**CWE:** CWE-494 – Download of Code Without Verification  
**CVSS:** 9.8  

**Location:** SensorWatch/src/main.cpp

**Evidence:**
```text
1994 if (!Update.begin(UPDATE_SIZE_UNKNOWN)) {
2003 if (Update.write(data, len) != len) {
2010 if (Update.end(true)) {
```
**Impact:**  
Firmware flashing occurs without cryptographic integrity checks. Attackers can upload malicious firmware that fully compromises system integrity.

**Recommendation:**  
Implement firmware signature verification (RSA/ECDSA) and reject any unauthenticated or tampered images.


## 3. Filesystem Format Operations

**Severity:** CRITICAL  
**CWE:** CWE-912 – Missing Authorization for Critical Function  
**CVSS:** 9.1  

**Location:** SensorWatch/src/main.cpp

**Evidence:**
```text
1276 if (LittleFS.format()) {
2157 if (LittleFS.format()) {
```

**Impact:**  
Allows complete filesystem wipe without authorization, leading to total data loss and device instability.

**Recommendation:** 
Restrict formatting operations to authenticated admin-level access.




## HIGH SEVERITY FINDINGS

## Category: Credential Issues

---

## 4. Hardcoded Credentials  

**Severity:** HIGH  
**CWE:** CWE-798  
**CVSS:** 7.5  

**Location:** `SensorWatch/src/main.cpp`

**Evidence:**
```text
1024 String wifiSSID = "guest";
1025 String wifiPassword = "password";
2429 File wifiFile = LittleFS.open("/wifi_config.json", "w");
```
**Impact:**  
Credentials stored in firmware can be extracted and abused.

**Recommendation:** 
Remove credential literals; use secure provisioning.


## 5. Hardcoded WiFi Credentials
### Rule: SensorWatch.cpp-hardcoded-wifi-credentials

**Severity:** HIGH  
**CWE:** CWE-259  
**CVSS:** 7.5  

**Location:** SensorWatch/src/main.cpp

**Evidence:**  
```text
1024 String wifiSSID = "guest";  
1025 String wifiPassword = "password";  
2231 String httpRequestData = "api_key=" + String(remoteApiKey) + "&data=" + jsonData;
```
**Impact:**  
Device compromise is trivial with default WiFi credentials.

**Recommendation:**  
Use secure setup flow and store credentials encrypted.



## 6. Hardcoded HTTP URL

**Severity:** HIGH  
**CWE:** CWE-319  
**CVSS:** 7.4  

**Location:** SensorWatch/src/main.cpp

**Evidence:**  
1026 const char* remoteServerName = "http://ecoforces.com/update_db.php";

**Impact:**  
HTTP exposes API key and sensor data to MITM attacks.

**Recommendation:**  
Switch to HTTPS and validate the server certificate.



## Category: Filesystem Controls
## 7. File Delete Operations


**Severity:** HIGH  
**CWE:** CWE-732  
**CVSS:** 7.8  

**Location:** SensorWatch/src/main.cpp

**Evidence:**  
2138 if (LittleFS.exists(path)) LittleFS.remove(path);  
2955 if (LittleFS.remove(filename)) {

**Impact:**  
Attackers can delete user or system configuration files.

**Recommendation:**  
Require authentication for delete operations.


## 8. File Write Operations
### Rule: SensorWatch.ota-fs-open-write

**Severity:** HIGH  
**CWE:** CWE-73  
**CVSS:** 7.5  

**Location:** SensorWatch/src/main.cpp

**Evidence:**  
1264 File file = LittleFS.open("/labels.json", "w");  
2303 File file = LittleFS.open("/data.json", "w");  
2312 File file = LittleFS.open("/data.json", "a");  
2429 File wifiFile = LittleFS.open("/wifi_config.json", "w");

**Impact:**  
Potential unauthorized file overwrite or tampering.

**Recommendation:**  
Enforce authentication and sanitize file paths.










15. Memory Allocation via new
Rule: SensorWatch.cpp-new-no-delete

Severity: LOW
CWE: CWE-401
CVSS: 3.1

Location: SensorWatch/src/main.cpp

Evidence:
```text
(Semgrep flagged 11 uses of 'new')
```
Impact:
Potential memory leaks.

Recommendation:
Use RAII / smart pointers.

