# Lab Write-Up: Brute Force Attack
**Date:** 2026-09-01  
**Author:** Adinath Dubile  
**Lab Environment:** DVWA (Damn Vulnerable Web Application)  
**Difficulty Levels Tested:** Low | Medium | High  
**Category:** Authentication  
**OWASP Top 10:** A07:2021 – Identification and Authentication Failures  

---

## 1. Introduction

Brute force attacks involve systematically attempting every possible combination of username and password until the correct credentials are found. Without rate limiting, account lockout, or CAPTCHA, web login forms are highly susceptible to automated credential stuffing and brute force attacks.

**Goal of this lab:** Use Burp Suite Intruder and Hydra to brute force the DVWA login form at all security levels and recover valid credentials.

---

## 2. Environment Setup

| Component        | Details                              |
|------------------|--------------------------------------|
| **Target App**   | DVWA – Brute Force Module            |
| **Attacker OS**  | Kali Linux                           |
| **Tools Used**   | Burp Suite, Hydra, SecLists          |
| **Network**      | Local lab (NAT/Host-Only)            |
| **Wordlists**    | `/usr/share/seclists/Passwords/Common-Credentials/` |

---

## 3. Vulnerability Overview

### What is Brute Force?

The DVWA brute force module simulates a vulnerable login form at `/vulnerabilities/brute/`. The form accepts a username and password via GET parameters.

**Why it's vulnerable:**
- No rate limiting
- No account lockout
- No CAPTCHA
- No multi-factor authentication
- Error messages confirm valid usernames (username enumeration)

---

## 4. Exploitation

### 4.1 Low Security Level

**Setup:** Navigate to DVWA → Brute Force. Set security to **Low**.

#### Step 1 – Understand the Login Request

1. Enter any test credentials (e.g., `admin:test`)
2. Intercept in Burp Suite

**GET request captured:**
```
GET /dvwa/vulnerabilities/brute/?username=admin&password=test&Login=Login HTTP/1.1
Host: localhost
Cookie: PHPSESSID=abc123; security=low
```

**Failed login response contains:**
```
Username and/or password incorrect.
```

**Successful login response contains:**
```
Welcome to the password protected area admin
```

#### Step 2 – Burp Suite Intruder Attack

1. Send the captured request to **Intruder** (Ctrl+I)
2. Go to **Positions** tab → Click "Clear §"
3. Highlight the `password` value and click "Add §":
   ```
   GET /dvwa/vulnerabilities/brute/?username=admin&password=§test§&Login=Login
   ```
4. Attack Type: **Sniper**

5. Go to **Payloads** tab:
   - Payload Type: **Simple list**
   - Load wordlist: `Pwdb_top-10000.txt`

6. Go to **Options** → **Grep - Match**:
   - Add string: `Welcome to the password protected area`

7. Click **Start Attack**

**Results:** Sort by Response Length or the grep match column. The correct password produces a longer response with the welcome message.

**Found:** `admin:password`

#### Step 3 – Username + Password Brute Force (Cluster Bomb)

For unknown usernames:
1. Positions: Mark both `username` and `password` as payload positions
2. Attack Type: **Cluster Bomb**
3. Set two payload lists:
   - Payload 1 (usernames): `unix_users.txt`
   - Payload 2 (passwords): `Pwdb_top-10000.txt`

> **Note:** Cluster bomb generates a cartesian product — large wordlists will produce many requests.

#### Step 4 – Hydra (Alternative)

```bash
hydra -l admin -P /usr/share/seclists/Passwords/Common-Credentials/Pwdb_top-10000.txt \
  localhost http-get-form \
  "/dvwa/vulnerabilities/brute/:username=^USER^&password=^PASS^&Login=Login:Username and/or password incorrect" \
  -t 10 -V
```

**Output:**
```
[80][http-get-form] host: localhost   login: admin   password: password
```

---

### 4.2 Medium Security Level

**Setup:** Set security to **Medium**.

**Observation:** A ~2-second sleep is added after a failed login attempt, slowing brute force attempts significantly.

#### Step 1 – Same Attack, Slower

The Burp Suite Intruder or Hydra attack from Low level still works — it just takes longer.

**Mitigation impact:** With a 2-second delay per attempt and a 10,000-word list, the attack would take ~5.5 hours. In a real engagement, this is a medium protection.

#### Step 2 – Optimize with Smaller Wordlists

Use a highly targeted wordlist:
```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
  localhost http-get-form \
  "/dvwa/vulnerabilities/brute/:username=^USER^&password=^PASS^&Login=Login:Username and/or password incorrect" \
  -t 1 -V
```

> Use `-t 1` (one thread) to respect the delay timing.

---

### 4.3 High Security Level

**Setup:** Set security to **High**.

**Observation:** An anti-CSRF **user_token** is added to each login attempt — the token changes after every request.

**Vulnerable request:**
```
GET /dvwa/vulnerabilities/brute/?username=admin&password=test&Login=Login&user_token=abc123def456 HTTP/1.1
```

Each failed request generates a new unique token for the next attempt.

#### Step 1 – Burp Suite Macro Approach

The token must be extracted from each response and used in the next request.

1. Go to **Settings → Sessions → Macros**
2. Click **Add** → Record a macro:
   - Request: `GET /dvwa/vulnerabilities/brute/` (the page load that returns the form with the token)
3. Click **Configure item** on the recorded GET request:
   - Custom parameter: `user_token`
   - Extract: From HTML attribute → `input[name="user_token"]` → `value`

4. Go to **Settings → Sessions → Session handling rules** → Add rule:
   - Action: **Run a macro**
   - Select the macro you created
   - Scope: Restrict to URL `/dvwa/vulnerabilities/brute/`

5. Set up Intruder attack:
   - Attack type: **Sniper** on `password` only
   - Payload: password wordlist
   - In Resource Pool → set **Maximum concurrent requests: 1** (critical — prevents token reuse)

6. Run the attack. Each request will now automatically:
   - Fetch a fresh page and extract the token
   - Use that token in the brute force request

**Result:** Password found even at High security level.

---

## 5. Findings Summary

| Security Level | Vulnerable | Tool Used          | Protection        | Time Estimate |
|----------------|------------|--------------------|-------------------|---------------|
| Low            | ✅ Yes     | Burp Intruder/Hydra | None              | Seconds       |
| Medium         | ✅ Yes     | Burp Intruder/Hydra | 2s delay          | Hours         |
| High           | ✅ Yes     | Burp Intruder+Macro | Anti-CSRF token   | Hours         |

---

## 6. Impact Assessment

| Category       | Rating   |
|----------------|----------|
| **Severity**   | High     |
| **CVSS Score** | 7.5      |
| **CIA Impact** | Confidentiality: High, Integrity: High, Availability: Medium |

**Business Impact:**
- Full account takeover for any user with a weak password
- Access to admin dashboard and sensitive data
- Pivot to other vulnerabilities once authenticated
- Password reuse risk across other systems

---

## 7. Remediation Recommendations

| Fix                            | Description                                                       |
|--------------------------------|-------------------------------------------------------------------|
| **Account Lockout**            | Lock account after 5-10 failed attempts                          |
| **Rate Limiting**              | Limit login attempts per IP/minute using fail2ban or middleware  |
| **CAPTCHA**                    | Implement reCAPTCHA after 3 failed attempts                      |
| **Multi-Factor Authentication**| Require OTP or authenticator app                                 |
| **Strong Password Policy**     | Minimum 12 chars, complexity requirements                        |
| **Security Headers**           | X-Frame-Options, HSTS to prevent credential theft                |
| **Logging & Alerting**         | Alert on multiple failed login attempts from same IP             |

---

## 8. References

- [OWASP Testing for Brute Force](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/04-Authentication_Testing/03-Testing_for_Weak_Lock_Out_Mechanism)
- [PortSwigger – Brute Force Attacks](https://portswigger.net/web-security/authentication/password-based)
- [DVWA Brute Force Walkthrough – ch3ngo](https://ch3ngo.github.io/posts/brute-force_dvwa_walkthrough/)
- [DVWA Brute Force Tutorial – Medium](https://medium.com/@dannybeton/dvwa-brute-force-tutorial-low-security-463880d53e50)
- [SecLists Wordlists](https://github.com/danielmiessler/SecLists)

---

## 9. Proof of Concept (PoC) Summary

```
Target:    DVWA – Brute Force Login Module
Tool:      Burp Suite Intruder (Sniper, Cluster Bomb) + Hydra
Wordlist:  Pwdb_top-10000.txt (SecLists)
Result:    admin:password cracked in < 60 seconds at Low level
Severity:  HIGH (CVSS 7.5)
```

---

*Write-up created for portfolio/educational purposes only. All testing performed in an isolated lab environment.*
