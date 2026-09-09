# Lab Write-Up: Weak Session IDs
**Date:** 2026-09-09  
**Author:** Adinath Dubile  
**Lab Environment:** DVWA (Damn Vulnerable Web Application)  
**Difficulty Levels Tested:** Low | Medium | High  
**Category:** Broken Authentication  
**OWASP Top 10:** A07:2021 – Identification and Authentication Failures  

---

## 1. Introduction

Weak Session IDs occur when web applications generate session tokens using predictable or guessable algorithms. Session tokens are the keys to a user's authenticated session — if an attacker can predict or enumerate them, they can hijack authenticated sessions without knowing a password.

**Goal of this lab:** Analyze how DVWA generates session tokens at each security level, predict the next token, and demonstrate session hijacking.

---

## 2. Environment Setup

| Component        | Details                              |
|------------------|--------------------------------------|
| **Target App**   | DVWA – Weak Session IDs Module       |
| **Attacker OS**  | Kali Linux                           |
| **Tools Used**   | Browser, Burp Suite, Burp Sequencer, John the Ripper, hashid |
| **Network**      | Local lab (NAT/Host-Only)            |

---

## 3. Vulnerability Overview

### What are Session IDs?

Session IDs are unique tokens assigned to users after login to maintain state across HTTP requests. A weak session ID is one that:
- Is sequential (1, 2, 3...)
- Is based on predictable data (timestamps)
- Uses weak hashing of predictable data (MD5 of a counter)
- Is too short to be cryptographically secure

**Attack Flow:**
```
1. Attacker observes their own session ID
2. Identifies the generation pattern
3. Predicts another valid session ID (e.g., admin's)
4. Sets that ID in their browser
5. Access granted as that user (Session Hijacking)
```

---

## 4. Exploitation

### 4.1 Low Security Level

**Setup:** Navigate to DVWA → Weak Session IDs. Set security to **Low**.

#### Step 1 – Observe Token Generation

1. Click "Generate" to create a new session ID
2. Open Burp Suite → HTTP history
3. Find the response `Set-Cookie` header:

**First click:**
```
Set-Cookie: dvwaSession=1; Path=/dvwa/
```

**Second click:**
```
Set-Cookie: dvwaSession=2; Path=/dvwa/
```

**Third click:**
```
Set-Cookie: dvwaSession=3; Path=/dvwa/
```

#### Step 2 – Pattern Analysis

**Pattern:** Sequential integer — `1, 2, 3, 4...`

This is completely predictable. An attacker who observes their own session ID of `15` can try `16` (admin's next session), or `1` (first user logged in).

#### Step 3 – Session Hijacking Demo

1. Open Browser A (victim), generate session → Cookie: `dvwaSession=5`
2. Open Browser B (attacker), manually set cookie:
   - Browser DevTools → Application → Cookies → `dvwaSession` = `5`
3. Navigate to a DVWA page

**Result:** Browser B is now authenticated as the victim's session. ✅

---

### 4.2 Medium Security Level

**Setup:** Set security to **Medium**.

#### Step 1 – Observe Token

**First click:**
```
Set-Cookie: dvwaSession=1753000000; Path=/dvwa/
```

**Second click (a few seconds later):**
```
Set-Cookie: dvwaSession=1753000005; Path=/dvwa/
```

#### Step 2 – Pattern Analysis

**Pattern:** Unix epoch timestamp (seconds since Jan 1, 1970)

The token is the Unix timestamp of when the button was clicked.

#### Step 3 – Verify Timestamp Theory

```bash
date -d @1753000000
# Mon Jul 20 2026 (example)
```

This matches the time the button was clicked — confirmed.

#### Step 4 – Predict a Session ID

If we know when a user logged in (e.g., from server logs, error messages, or social engineering), we can calculate their session ID:

```bash
date -d "2026-08-20 09:30:00" +%s
# 1755733800
```

Try `dvwaSession=1755733800` for session hijacking.

**Brute force window:** With second-level precision, there are only 86,400 possible tokens per day — easily brute-forceable.

---

### 4.3 High Security Level

**Setup:** Set security to **High**.

#### Step 1 – Observe Token

**First click:**
```
Set-Cookie: dvwaSession=e4da3b7fbbce2345d7772b0674a318d5; Path=/dvwa/
```

This is a 32-character hex string — looks like an MD5 hash.

#### Step 2 – Identify Hash Type

```bash
hashid e4da3b7fbbce2345d7772b0674a318d5
```

**Output:** `MD5`

#### Step 3 – Crack the Hash (John the Ripper)

Collect multiple tokens and save to `session_hashes.txt`:
```
e4da3b7fbbce2345d7772b0674a318d5
1679091c5a880faf6fb5e6087eb1b2dc
8f14e45fceea167a5a36dedd4bea2543
```

```bash
john --format=Raw-MD5 session_hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**Result:**
```
e4da3b7fbbce2345d7772b0674a318d5 → 1
1679091c5a880faf6fb5e6087eb1b2dc → 2
8f14e45fceea167a5a36dedd4bea2543 → 5
```

The hashes are just MD5 of sequential numbers: `MD5(1)`, `MD5(2)`, `MD5(5)`

#### Step 4 – Confirm and Predict

```bash
echo -n "3" | md5sum
# eccbc87e4b5ce2fe28308fd9f2a7baf3
```

Try `dvwaSession=eccbc87e4b5ce2fe28308fd9f2a7baf3` — this would be session #3.

#### Step 5 – Burp Sequencer Analysis

For a thorough randomness analysis:
1. Burp → HTTP history → Find a response with `Set-Cookie: dvwaSession=...`
2. Right-click → **Send to Sequencer**
3. Configure: **Cookie** → `dvwaSession`
4. Click **Start live capture** (collect 100+ tokens)
5. Analyze → Check entropy score

**Expected result at Low/Medium:** Very low entropy — easily predictable.

---

## 5. Additional Cookie Flags Observed (High Level)

```
Set-Cookie: dvwaSession=e4da3b7fbbce...; Path=/dvwa/; HttpOnly; Secure
```

While these flags are present at High level, they don't fix the predictability problem:
- `HttpOnly` — prevents JS from reading the cookie, but we're predicting it, not stealing it via XSS
- `Secure` — ensures HTTPS transmission only
- Neither flag addresses the weak generation algorithm

---

## 6. Findings Summary

| Security Level | Algorithm         | Predictable | Attack Vector        |
|----------------|-------------------|-------------|----------------------|
| Low            | Sequential integer | ✅ Yes      | Direct enumeration   |
| Medium         | Unix timestamp     | ✅ Yes      | Time-window brute force |
| High           | MD5(sequential int)| ✅ Yes      | Hash cracking        |

---

## 7. Impact Assessment

| Category       | Rating   |
|----------------|----------|
| **Severity**   | High     |
| **CVSS Score** | 8.0      |
| **CIA Impact** | Confidentiality: High, Integrity: High, Availability: Medium |

**Business Impact:**
- Session hijacking → account takeover
- Privilege escalation (predict admin session)
- Access to private data, financial operations
- Bypasses password protection completely

---

## 8. Remediation Recommendations

| Fix                             | Description                                                     |
|---------------------------------|-----------------------------------------------------------------|
| **CSPRNG**                      | Use cryptographically secure random number generators          |
| **Sufficient Token Length**     | Minimum 128-bit (16 bytes) random tokens                       |
| **No Predictable Seeds**        | Don't use timestamps, counters, or usernames as session seeds  |
| **Re-generate After Login**     | Generate a new session ID after every successful authentication |
| **Short Expiry + Rotation**     | Expire sessions after inactivity and rotate regularly          |
| **HttpOnly + Secure + SameSite**| Always set all three cookie security flags                     |
| **PHP Session Security**        | Use `session_regenerate_id(true)` after login                  |

**Secure Code (PHP):**
```php
// Vulnerable (Low level equivalent)
$_COOKIE['dvwaSession'] = $_COOKIE['dvwaSession'] + 1;

// Secure – use PHP's built-in CSPRNG
session_start();
$session_token = bin2hex(random_bytes(32)); // 256-bit entropy
setcookie('dvwaSession', $session_token, [
    'httponly' => true,
    'secure'   => true,
    'samesite' => 'Strict'
]);
```

---

## 9. References

- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [PortSwigger – Session Tokens](https://portswigger.net/web-security/authentication/other-mechanisms)
- [DVWA Weak IDs Walkthrough – ch3ngo](https://ch3ngo.github.io/posts/weak-ids_dvwa_walkthrough/)
- [DVWA Weak Session IDs – cspanias](https://cspanias.github.io/posts/DVWA-Weak-Session-IDs)

---

## 10. Proof of Concept (PoC) Summary

```
Target:    DVWA – Weak Session IDs Module
Low:       dvwaSession = sequential int (1, 2, 3...)
Medium:    dvwaSession = Unix timestamp (predictable to the second)
High:      dvwaSession = MD5(sequential int) → cracked by John the Ripper
Severity:  HIGH (CVSS 8.0)
```

---

*Write-up created for portfolio/educational purposes only. All testing performed in an isolated lab environment.*
