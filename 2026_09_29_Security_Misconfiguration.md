# Lab Write-Up: Security Misconfiguration
**Date:** 2026-09-29  
**Author:** Adinath Dubile  
**Lab Environment:** DVWA (Damn Vulnerable Web Application)  
**Difficulty Levels Tested:** Application-Wide Analysis  
**Category:** Security Misconfiguration  
**OWASP Top 10:** A05:2021 – Security Misconfiguration  

---

## 1. Introduction

Security Misconfiguration is the most commonly found vulnerability in web applications. It occurs when security settings are not defined, implemented, or maintained correctly — including default credentials, verbose error messages, unnecessary features enabled, missing security headers, and improper access controls. DVWA itself is a misconfigured application by design, making it an ideal platform to study these issues.

**Goal of this lab:** Identify and document all security misconfigurations present in the default DVWA installation and demonstrate the impact of each.

---

## 2. Environment Setup

| Component        | Details                              |
|------------------|--------------------------------------|
| **Target App**   | DVWA (full application)              |
| **Attacker OS**  | Kali Linux                           |
| **Tools Used**   | Nikto, Browser (DevTools), Burp Suite, curl |
| **Network**      | Local lab (NAT/Host-Only)            |

---

## 3. Discovery: Automated Scanning

### Step 1 – Nikto Web Scanner

```bash
nikto -h http://localhost/dvwa/
```

**Key findings from Nikto:**
```
+ Server: Apache/2.4.57 (Debian)
+ /dvwa/phpinfo.php: Output from the phpinfo() function was found.
+ /dvwa/setup.php: Setup script is publicly accessible.
+ /dvwa/config/: Directory is browsable.
+ X-Frame-Options header is not present.
+ X-Content-Type-Options header is not set.
+ The X-XSS-Protection header is not set.
+ Retrieved x-powered-by header: PHP/8.2.7
```

### Step 2 – Enumerate Security Headers

```bash
curl -I http://localhost/dvwa/ 2>/dev/null | grep -i "header\|server\|x-\|strict\|content-security\|set-cookie"
```

**Output:**
```
Server: Apache/2.4.57 (Debian)
X-Powered-By: PHP/8.2.7
Set-Cookie: PHPSESSID=abc123; path=/
```

**Missing headers:**
```
❌ Strict-Transport-Security
❌ Content-Security-Policy
❌ X-Frame-Options
❌ X-Content-Type-Options
❌ Referrer-Policy
❌ Permissions-Policy
```

---

## 4. Vulnerability Findings

### 4.1 Default Credentials

DVWA ships with default admin credentials that are widely known and documented.

**Default Login:**
```
Username: admin
Password: password
```

**Test:**
```bash
curl -c cookies.txt -d "username=admin&password=password&Login=Login" \
  http://localhost/dvwa/login.php -L -v 2>&1 | grep "Welcome"
```

**Output:** Successful login → `Welcome admin`

**Risk:** Any attacker can immediately gain admin access to a DVWA installation left at defaults.

---

### 4.2 phpinfo.php Exposed

The `phpinfo.php` file reveals the full PHP configuration to anyone:

```
http://localhost/dvwa/phpinfo.php
```

**Sensitive information disclosed:**
```
PHP Version: 8.2.7
Document Root: /var/www/html
_SERVER["DB_PASSWORD"] = p@ssw0rd   (if set as env var)
allow_url_include: On               ← enables RFI attacks
allow_url_fopen: On
display_errors: On                  ← verbose errors
Loaded Extensions: mysqli, curl, gd...
```

---

### 4.3 Setup Page Publicly Accessible

```
http://localhost/dvwa/setup.php
```

The database setup/reset page is accessible without authentication (or with just default creds). An attacker can:
- Reset the entire DVWA database
- Reinitialize all data
- Identify database structure

---

### 4.4 Verbose Error Messages

DVWA displays full PHP error messages including file paths:

**Trigger a PHP error by passing invalid input to any module.**

**Output:**
```
Warning: include(/var/www/html/dvwa/vulnerabilities/fi/../../INVALIDPATH): 
failed to open stream: No such file or directory in 
/var/www/html/dvwa/vulnerabilities/fi/index.php on line 36
```

**Information disclosed:**
- Full server file path: `/var/www/html/dvwa/`
- File structure and line numbers
- PHP version and function names

---

### 4.5 Directory Listing Enabled

```
http://localhost/dvwa/config/
http://localhost/dvwa/hackable/uploads/
```

**Output:** Browser shows a full directory listing — all files are visible and downloadable.

**Sensitive files visible:**
```
/dvwa/config/config.inc.php.dist   ← default config template
/dvwa/hackable/uploads/shell.php   ← if previously uploaded!
```

---

### 4.6 Missing Security Headers

Test each header:

```bash
# Check for Clickjacking protection
curl -I http://localhost/dvwa/ | grep X-Frame-Options
# Output: (empty) → Clickjacking possible

# Check for MIME sniffing protection
curl -I http://localhost/dvwa/ | grep X-Content-Type-Options
# Output: (empty) → MIME confusion possible

# Check for XSS protection
curl -I http://localhost/dvwa/ | grep X-XSS-Protection
# Output: (empty) → No header-level XSS filter

# Check CSP
curl -I http://localhost/dvwa/ | grep Content-Security-Policy
# Output: (empty) → No CSP restrictions
```

---

### 4.7 Cookies Without Security Flags

```
Set-Cookie: PHPSESSID=abc123; path=/
```

Missing flags:
- **No `HttpOnly`** → Cookie accessible via JavaScript (XSS can steal it)
- **No `Secure`** → Cookie sent over HTTP (network sniffing)
- **No `SameSite`** → Cookie sent in cross-origin requests (CSRF)

---

### 4.8 Server Version Disclosure

```
Server: Apache/2.4.57 (Debian)
X-Powered-By: PHP/8.2.7
```

Exact version numbers help attackers search for known CVEs:
```bash
searchsploit apache 2.4.57
searchsploit php 8.2
```

---

### 4.9 HTTP (No HTTPS)

DVWA runs on HTTP by default — all traffic including passwords and session cookies is transmitted in cleartext.

```bash
# Verify no HTTPS redirect
curl -I http://localhost/dvwa/ | grep "Location\|Strict-Transport"
# Output: (empty) → No HTTPS enforcement
```

---

## 5. Findings Summary

| Misconfiguration              | Severity | Impact                           |
|-------------------------------|----------|----------------------------------|
| Default credentials           | Critical | Immediate admin access           |
| phpinfo.php exposed           | High     | Server/config info disclosure    |
| Setup page accessible         | High     | Database reset by anyone         |
| Verbose error messages        | Medium   | Path/structure disclosure        |
| Directory listing enabled     | Medium   | File structure exposure          |
| Missing security headers      | Medium   | XSS, Clickjacking, MIME attacks  |
| No HttpOnly/Secure cookies    | High     | Session hijacking via XSS/sniff  |
| Server version disclosure     | Low      | CVE research assistance          |
| HTTP only (no HTTPS)          | High     | Full traffic interception        |

---

## 6. Impact Assessment

| Category       | Rating   |
|----------------|----------|
| **Severity**   | Critical (cumulative) |
| **CVSS Score** | 9.8 (worst finding)  |
| **CIA Impact** | Confidentiality: High, Integrity: High, Availability: Medium |

---

## 7. Remediation Recommendations

**Security Headers (Apache):**
```apache
# Add to /etc/apache2/conf-available/security.conf
Header always set X-Frame-Options "DENY"
Header always set X-Content-Type-Options "nosniff"
Header always set X-XSS-Protection "1; mode=block"
Header always set Referrer-Policy "strict-origin-when-cross-origin"
Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
Header always set Content-Security-Policy "default-src 'self'"
Header unset X-Powered-By
ServerTokens Prod
ServerSignature Off
```

**PHP hardening (`php.ini`):**
```ini
display_errors = Off
expose_php = Off
allow_url_include = Off
open_basedir = /var/www/html/dvwa
```

**Cookie security (PHP):**
```php
ini_set('session.cookie_httponly', 1);
ini_set('session.cookie_secure', 1);
ini_set('session.cookie_samesite', 'Strict');
```

**Additional steps:**
- Change default credentials immediately after installation
- Remove or restrict access to `phpinfo.php`, `setup.php`
- Disable directory listing in Apache: `Options -Indexes`
- Deploy behind HTTPS with valid TLS certificate
- Implement a WAF (ModSecurity)

---

## 8. References

- [OWASP Security Misconfiguration](https://owasp.org/Top10/A05_2021-Security_Misconfiguration/)
- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/)
- [Mozilla Observatory](https://observatory.mozilla.org/)
- [SecurityHeaders.com](https://securityheaders.com/)
- [Nikto Web Scanner](https://github.com/sullo/nikto)

---

## 9. Proof of Concept (PoC) Summary

```
Target:    DVWA – Default Installation (All Security Levels)
Finding 1: Default creds admin:password → Instant admin access
Finding 2: phpinfo.php exposed → Full server config disclosed
Finding 3: No HttpOnly on PHPSESSID → XSS steals session trivially
Finding 4: 6 security headers missing → Clickjacking, MIME, XSS possible
Finding 5: Directory listing → Uploaded shells visible at /hackable/uploads/
Severity:  CRITICAL — multiple high-impact misconfigurations chained
```

---

*Write-up created for portfolio/educational purposes only. All testing performed in an isolated lab environment.*
