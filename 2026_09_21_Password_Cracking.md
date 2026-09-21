# Lab Write-Up: Password Cracking (Hash Analysis)
**Date:** 2026-09-21  
**Author:** Adinath Dubile  
**Lab Environment:** DVWA (Damn Vulnerable Web Application)  
**Difficulty Levels Tested:** Post-Exploitation  
**Category:** Cryptographic Failures  
**OWASP Top 10:** A02:2021 – Cryptographic Failures  

---

## 1. Introduction

After successfully exploiting SQL Injection in DVWA to extract password hashes, the next step in a real penetration test is cracking those hashes offline. This write-up focuses on hash identification, cracking techniques using John the Ripper and Hashcat, and understanding the security implications of using weak (unsalted MD5) password hashing.

**Goal of this lab:** Take the password hashes extracted from the DVWA database (via SQL injection) and crack them using offline tools to recover plaintext passwords.

---

## 2. Environment Setup

| Component        | Details                                          |
|------------------|--------------------------------------------------|
| **Source**       | Hashes from DVWA SQL Injection write-up          |
| **Attacker OS**  | Kali Linux                                       |
| **Tools Used**   | hashid, John the Ripper, Hashcat                 |
| **Wordlists**    | `/usr/share/wordlists/rockyou.txt`, SecLists     |
| **Hardware**     | CPU-based cracking (lab environment)             |

---

## 3. Extracted Hashes

Hashes obtained from the DVWA `users` table via SQL injection:

| Username | Hash                               |
|----------|------------------------------------|
| admin    | `5f4dcc3b5aa765d61d8327deb882cf99` |
| gordonb  | `e99a18c428cb38d5f260853678922e03` |
| 1337     | `8d3533d75ae2c3966d7e0d4fcc69216b` |
| pablo    | `0d107d09f5bbe40cade3de5c71e9e9b7` |
| smithy   | `5f4dcc3b5aa765d61d8327deb882cf99` |

---

## 4. Step-by-Step Hash Cracking

### Step 1 – Hash Identification

```bash
hashid 5f4dcc3b5aa765d61d8327deb882cf99
```

**Output:**
```
Analyzing '5f4dcc3b5aa765d61d8327deb882cf99'
[+] MD2
[+] MD5          <-- Most likely
[+] MD4
[+] Double MD5
...
```

**Confirm it's MD5:** 32-character hex string = MD5.

```bash
# Also check with hash-identifier
hash-identifier
> 5f4dcc3b5aa765d61d8327deb882cf99
```

**Result:** MD5 (unsalted)

---

### Step 2 – Online Lookup (Fastest Method)

Before running CPU-intensive cracking, check online rainbow tables:

**CrackStation:** https://crackstation.net/
- Paste all 5 hashes
- **Results returned instantly:**

```
5f4dcc3b5aa765d61d8327deb882cf99 → password
e99a18c428cb38d5f260853678922e03 → abc123
8d3533d75ae2c3966d7e0d4fcc69216b → charley
0d107d09f5bbe40cade3de5c71e9e9b7 → letmein
5f4dcc3b5aa765d61d8327deb882cf99 → password
```

All 5 cracked immediately because these are common passwords in rainbow tables.

---

### Step 3 – John the Ripper (Dictionary Attack)

Save hashes to `hashes.txt`:
```
5f4dcc3b5aa765d61d8327deb882cf99
e99a18c428cb38d5f260853678922e03
8d3533d75ae2c3966d7e0d4fcc69216b
0d107d09f5bbe40cade3de5c71e9e9b7
```

**Run John:**
```bash
john --format=Raw-MD5 hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**Output:**
```
Using default input encoding: UTF-8
Loaded 4 password hashes with no different salts (Raw-MD5 [MD5 256/256 AVX2 8x3])
Press 'q' or Ctrl-C to abort, almost any other key for status
password         (?)
abc123           (?)
charley          (?)
letmein          (?)
4g 0:00:00:01 DONE 2.040g/s 8532Kp/s 8532Kc/s
```

**Show cracked results:**
```bash
john --format=Raw-MD5 hashes.txt --show
```

**Output:**
```
?:password
?:abc123
?:charley
?:letmein
4 password hashes cracked, 0 left
```

---

### Step 4 – John the Ripper (Rules-Based Attack)

For more complex passwords, use mutation rules:
```bash
john --format=Raw-MD5 hashes.txt \
  --wordlist=/usr/share/wordlists/rockyou.txt \
  --rules=best64
```

---

### Step 5 – Hashcat (GPU-Accelerated)

For faster cracking (especially with GPU):

**Dictionary attack:**
```bash
hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt
```

**Rule-based attack:**
```bash
hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

**Brute force (all 8-char alphanumeric):**
```bash
hashcat -m 0 hashes.txt -a 3 ?l?l?l?l?l?l?l?l
```

**Mode reference:**
```
-m 0     = MD5
-m 100   = SHA1
-m 1400  = SHA256
-m 1800  = SHA512crypt (Linux shadow)
-m 3200  = bcrypt (much harder!)
```

---

### Step 6 – Named Hashcat Results

Add usernames to the hash file (john format):
```
admin:5f4dcc3b5aa765d61d8327deb882cf99
gordonb:e99a18c428cb38d5f260853678922e03
1337:8d3533d75ae2c3966d7e0d4fcc69216b
pablo:0d107d09f5bbe40cade3de5c71e9e9b7
smithy:5f4dcc3b5aa765d61d8327deb882cf99
```

```bash
john --format=Raw-MD5 named_hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
john --show named_hashes.txt
```

**Output:**
```
admin:password
gordonb:abc123
1337:charley
pablo:letmein
smithy:password
```

---

## 5. Hash Comparison — Why MD5 Is Broken

| Algorithm    | Speed (CPU)    | Crackable? | Used for passwords? |
|--------------|----------------|------------|---------------------|
| MD5          | ~1 billion/sec | ✅ Yes     | ❌ Never            |
| SHA1         | ~700M/sec      | ✅ Yes     | ❌ Never            |
| SHA256       | ~400M/sec      | ✅ Yes     | ❌ Never            |
| bcrypt(12)   | ~100/sec       | ⚠️ Hard    | ✅ Acceptable       |
| Argon2id     | ~10/sec        | ❌ Impractical | ✅ Recommended  |

**Unsalted MD5 at 1 billion attempts/sec:** An 8-character lowercase password cracks in under 1 second.

---

## 6. Findings Summary

| Hash                               | Username | Cracked Password | Method          |
|------------------------------------|----------|-----------------|-----------------|
| `5f4dcc3b5aa765d61d8327deb882cf99` | admin    | `password`      | Online lookup   |
| `e99a18c428cb38d5f260853678922e03` | gordonb  | `abc123`        | Online lookup   |
| `8d3533d75ae2c3966d7e0d4fcc69216b` | 1337     | `charley`       | John dictionary |
| `0d107d09f5bbe40cade3de5c71e9e9b7` | pablo    | `letmein`       | John dictionary |
| `5f4dcc3b5aa765d61d8327deb882cf99` | smithy   | `password`      | Duplicate of admin |

---

## 7. Impact Assessment

| Category       | Rating   |
|----------------|----------|
| **Severity**   | Critical |
| **CVSS Score** | 9.1      |
| **CIA Impact** | Confidentiality: High, Integrity: High, Availability: Medium |

**Business Impact:**
- Full account access for all 5 users including admin
- Password reuse — same passwords likely used on other systems
- Insider threat escalation
- Regulatory compliance failure (GDPR — inadequate data protection)

---

## 8. Remediation Recommendations

| Fix                               | Description                                                       |
|-----------------------------------|-------------------------------------------------------------------|
| **Use bcrypt or Argon2id**        | Use password-hashing algorithms designed for this purpose        |
| **Never Use MD5/SHA for Passwords** | These are fast algorithms — catastrophic for password storage  |
| **Add Salts**                     | Unique random salt per password defeats rainbow tables            |
| **Password Policy**               | Enforce minimum 12 characters, complexity, no common words       |
| **MFA**                           | Even cracked passwords useless without second factor             |
| **Have I Been Pwned API**         | Check passwords against known breach databases at registration   |

**Secure Password Hashing (PHP):**
```php
// Vulnerable (DVWA approach)
$hash = md5($password);

// Secure – PHP password_hash uses bcrypt by default
$hash = password_hash($password, PASSWORD_BCRYPT, ['cost' => 12]);

// Verify
if (password_verify($input_password, $hash)) {
    // Authenticated
}
```

---

## 9. References

- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [CrackStation](https://crackstation.net/)
- [Hashcat Documentation](https://hashcat.net/wiki/)
- [John the Ripper](https://www.openwall.com/john/)
- [Have I Been Pwned](https://haveibeenpwned.com/)

---

## 10. Proof of Concept (PoC) Summary

```
Source:    Hashes from DVWA SQL Injection (users table)
Algorithm: Unsalted MD5
Tools:     John the Ripper, Hashcat, CrackStation
Results:   100% of 5 hashes cracked (password, abc123, charley, letmein, password)
Time:      < 2 seconds (CrackStation), < 60 seconds (John)
Severity:  CRITICAL — demonstrates why MD5 must never store passwords
```

---

*Write-up created for portfolio/educational purposes only. All testing performed in an isolated lab environment.*
