# Lab Write-Up: JavaScript Attacks
**Date:** 2026-09-13  
**Author:** Adinath Dubile  
**Lab Environment:** DVWA (Damn Vulnerable Web Application)  
**Difficulty Levels Tested:** Low | Medium | High  
**Category:** Client-Side Security  
**OWASP Top 10:** A02:2021 – Cryptographic Failures / Client-Side Validation  

---

## 1. Introduction

JavaScript Attacks exploit the fact that client-side code is fully visible and modifiable by any user. Applications that rely on JavaScript for security-critical operations — such as token generation, input validation, or access control — are fundamentally flawed, because any user can read the logic, modify values, or bypass checks entirely.

**Goal of this lab:** Analyze the client-side JavaScript token generation logic in DVWA, reverse-engineer the token for each security level, and submit the word "success" to complete each challenge.

---

## 2. Environment Setup

| Component        | Details                              |
|------------------|--------------------------------------|
| **Target App**   | DVWA – JavaScript Module             |
| **Attacker OS**  | Kali Linux                           |
| **Tools Used**   | Browser (DevTools), Burp Suite, CyberChef, de4js |
| **Network**      | Local lab (NAT/Host-Only)            |

---

## 3. Vulnerability Overview

### Why is Client-Side Security Broken?

The golden rule of web security: **Never trust the client**.

Client-side JavaScript is:
- Fully readable by anyone (View Source / DevTools)
- Modifiable at runtime (browser console)
- Interceptable (Burp Suite proxy)
- Not a security boundary

DVWA's JavaScript challenge uses client-side token generation to "protect" the form. The goal is to understand why this fails at every level.

---

## 4. Exploitation

### 4.1 Low Security Level

**Setup:** Navigate to DVWA → JavaScript. Set security to **Low**.

#### Step 1 – Understand the Challenge

The page has a text field. Submit the word "success" and the challenge is complete — but the application validates a **token** alongside the submission.

#### Step 2 – Inspect the Source Code

Open DevTools → Sources → find `token.js` or inline `<script>`:

```javascript
function do_elsesomething(e) {
    document.getElementById("token").value = md5(rot13(document.getElementById("phrase").value));
}
```

**Token generation logic:**
1. Take the value from the `phrase` input field
2. Apply ROT13 encoding
3. Apply MD5 hash
4. Set the result as the `token` field value

#### Step 3 – Calculate the Token

**Input:** `success`

**Step A – ROT13 of "success":**
```
s → f
u → h  
c → p
c → p
e → r
s → f
s → f

ROT13("success") = "fhpprff"
```

**Step B – MD5 of "fhpprff":**
```bash
echo -n "fhpprff" | md5sum
# Output: 38581812b435834ebf84ebcc2c6424d0
```

Or use CyberChef:
1. Input: `success`
2. Operations: ROT13 → MD5

**Token:** `38581812b435834ebf84ebcc2c6424d0`

#### Step 4 – Submit via Burp

Intercept the form submission and set the token:

**POST body:**
```
phrase=success&token=38581812b435834ebf84ebcc2c6424d0&send=Submit
```

**Result:** "Well done!" ✅

---

### 4.2 Medium Security Level

**Setup:** Set security to **Medium**.

**Observation:** The JavaScript is now **minified** (all whitespace removed, variables shortened) to make it harder to read.

#### Step 1 – Find the JavaScript File

DevTools → Sources → find the external `.js` file (e.g., `token.js`)

**Minified code example:**
```javascript
function do_something(e){for(var t="",n=e.length-1;n>=0;n--)t+=e[n];return t}function token_part_3(t,y=document.getElementById("phrase").value){document.getElementById("token").value=sha256(document.getElementById("token").value+y)}function token_part_2(e="XX"){document.getElementById("token").value=sha256(e+do_something(document.getElementById("token").value))}function token_part_1(a,b){document.getElementById("token").value=do_something(document.getElementById("phrase").value)}
```

#### Step 2 – Pretty Print for Readability

DevTools → Open the JS file → Click `{}` (Pretty Print) at the bottom:

```javascript
function do_something(e) {
    for (var t = "", n = e.length - 1; n >= 0; n--)
        t += e[n];
    return t; // Reverses the string
}
function token_part_1(a, b) {
    document.getElementById("token").value = do_something(
        document.getElementById("phrase").value
    );
}
function token_part_2(e = "XX") {
    document.getElementById("token").value = sha256(
        e + do_something(document.getElementById("token").value)
    );
}
function token_part_3(t, y = document.getElementById("phrase").value) {
    document.getElementById("token").value = sha256(
        document.getElementById("token").value + y
    );
}
```

#### Step 3 – Analyze Token Generation Logic

The token is built in three parts:

1. **Part 1:** `token = reverse("success")` = `"sseccus"`
2. **Part 2:** `token = SHA256("XX" + reverse("sseccus"))` = `SHA256("XX" + "success")` = `SHA256("XXsuccess")`
3. **Part 3:** `token = SHA256(token + "success")`

**Wait — recalculate Part 2:**
- `do_something("sseccus")` = reverses `"sseccus"` = `"success"` again
- `token_part_2("XX")` = `SHA256("XX" + "success")` = `SHA256("XXsuccess")`

Hmm, let's re-read: `do_something` reverses the current token value (which at Part 2 is `"sseccus"`):
- `reverse("sseccus")` = `"success"` 
- `SHA256("XX" + "success")` = `SHA256("XXsuccess")`

**Part 3:**
- `SHA256(previous_token + "success")`

#### Step 4 – Use the Browser Debugger to Extract Token

The cleanest approach — let the browser compute it:

1. Open DevTools → Sources → find `token.js`
2. Set a **breakpoint** on the `token_part_3` function at the last line
3. In the browser console, manually set: `document.getElementById("phrase").value = "success"`
4. Trigger `token_part_1()`, then `token_part_2()`, then `token_part_3()`
5. At the breakpoint, read the final token value from: `document.getElementById("token").value`

**Or simply via console:**
```javascript
document.getElementById("phrase").value = "success";
token_part_1();
token_part_2();
token_part_3();
console.log(document.getElementById("token").value);
```

**Copy the resulting token and submit via Burp.**

**Result:** "Well done!" ✅

---

### 4.3 High Security Level

**Setup:** Set security to **High**.

**Observation:** The JavaScript is now **obfuscated** — variable names are replaced with meaningless characters, strings are encoded, and the logic is deliberately obscured.

#### Step 1 – Identify the Script

The script is typically in an external `.js` file. It may look like:

```javascript
var _0x2b4f=['getElementById','value','sha256','phrase','token','fromCharCode',...];
(function(_0x...){...})(...);
```

#### Step 2 – Deobfuscate

Use an online deobfuscator:
- **de4js:** https://de4js.kshift.me/
- **JStillery:** https://mindedsecurity.github.io/jstillery/

Paste the obfuscated code → click Deobfuscate.

**Output (cleaned up):**
```javascript
function generate_token() {
    var phrase = document.getElementById('phrase').value;
    var token = sha256(phrase + 'token');
    document.getElementById('token').value = token;
}
```

#### Step 3 – Use Burp + Local Script Override

A powerful technique: use Burp's **Match and Replace** to serve a modified (readable) version of the script:

1. Host the deobfuscated script on your Kali machine:
   ```bash
   python3 -m http.server 9000
   ```

2. In Burp → **Proxy → Match and Replace**:
   - Type: Response header
   - Match: original script URL
   - Replace: `http://192.168.x.x:9000/clean_token.js`

3. Add breakpoints in the clean version to extract the token.

#### Step 4 – Browser Console Approach

Even with obfuscation, all functions are available in the browser's global scope. In DevTools console:

```javascript
document.getElementById("phrase").value = "success";
// Trigger each token-building function (names visible in debugger)
generate_token();
console.log(document.getElementById("token").value);
```

**Submit the extracted token via Burp.**

**Result:** "Well done!" ✅

---

## 5. Key Takeaways

```
❌ Client-side validation = NOT security
❌ Minification ≠ obfuscation ≠ protection
❌ Token generation in JS = reversible by any user
✅ Security-critical operations must happen server-side
✅ Tokens must be generated server-side and verified server-side
```

---

## 6. Findings Summary

| Security Level | Protection      | Bypass Method                        | Time      |
|----------------|-----------------|--------------------------------------|-----------|
| Low            | None            | Read JS, manual calculation          | Seconds   |
| Medium         | Minification    | Pretty-print + console execution     | Minutes   |
| High           | Obfuscation     | Deobfuscation tools + debugger       | Minutes   |

---

## 7. Impact Assessment

| Category       | Rating   |
|----------------|----------|
| **Severity**   | Medium   |
| **CVSS Score** | 6.5      |
| **CIA Impact** | Confidentiality: Medium, Integrity: High, Availability: Low |

**Business Impact:**
- Bypass of client-side access controls
- Authentication bypass if login tokens are generated client-side
- License/feature restriction bypasses
- Form validation bypass enabling injection attacks

---

## 8. Remediation Recommendations

| Fix                                   | Description                                              |
|---------------------------------------|----------------------------------------------------------|
| **Server-Side Token Generation**      | Generate and validate all security tokens on the server  |
| **Never Trust Client Input**          | Treat all client data as untrusted                       |
| **Server-Side Input Validation**      | Validate all inputs server-side, not in JS               |
| **Remove Business Logic from JS**     | Don't put any security-critical logic in client code     |
| **Signed Tokens (HMAC)**             | If tokens must be client-side, sign them with a server secret |

---

## 9. References

- [OWASP Client-Side Security](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/11-Client-Side_Testing/)
- [PortSwigger – DOM-Based Vulnerabilities](https://portswigger.net/web-security/dom-based)
- [DVWA JavaScript Walkthrough – ch3ngo](https://ch3ngo.github.io/posts/js-attacks_dvwa_walkthrough/)
- [DVWA Javascript – cspanias](https://cspanias.github.io/posts/DVWA-Javascript)
- [de4js Deobfuscator](https://de4js.kshift.me/)

---

## 10. Proof of Concept (PoC) Summary

```
Target:    DVWA – JavaScript Module
Low:       ROT13("success") → MD5 → token calculated and submitted
Medium:    Browser debugger used to extract computed token
High:      JS deobfuscated → token computation understood → submitted
Result:    "Well done!" at all 3 levels
Severity:  MEDIUM (CVSS 6.5) — escalates if pattern used for auth
```

---

*Write-up created for portfolio/educational purposes only. All testing performed in an isolated lab environment.*
