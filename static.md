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

No evidence is rewritten — evidence comes from Semgrep output exactly.

---

# 4. Detailed Findings by Rule (Industry Format)

---

## 4.1 Hardcoded Credentials  
### Rule: `SensorWatch.cpp-hardcoded-credential-literal`  
**Severity:** HIGH  
**CWE:** CWE-798 – Use of Hardcoded Credentials  
**CVSS:** 7.5 (High)

**Description:**  
Semgrep detected literal credential-like strings inside firmware code.

**Impact:**  
Attackers with access to the firmware can extract keys and gain unauthorized access to services.

**Evidence (from Semgrep):**
```text
1024┆ String wifiSSID = "guest";
1025┆ String wifiPassword = "password";
2429┆ File wifiFile = LittleFS.open("/wifi_config.json", "w");
