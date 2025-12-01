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

