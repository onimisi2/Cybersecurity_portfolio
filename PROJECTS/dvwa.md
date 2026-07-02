# DVWA Penetration Test — Lab Report

**Target:** DVWA (Damn Vulnerable Web Application)  
**Environment:** Docker container on Kali Linux (VMware Workstation)  
**Date:** July 2026  
**Tester:** [stephen smith]  
**Status:** Completed

---

## Executive Summary

A full penetration test was conducted against a locally hosted DVWA instance. 
Starting with zero credentials, the assessment resulted in full administrative 
access to the web application and direct database access through a chain of 
misconfigurations and weak credential practices.

---

## Methodology

Reconnaissance → Enumeration → Exploitation → Credential Access → Authentication

---

## Findings

### Finding 1 — Open Port and Service Version Disclosure
**Severity:** Low  
**Tool:** Nmap  
**Command:**
```bash
nmap -sV 127.0.0.1
```
**Result:** Port 80 open running Apache HTTP Server 2.4.25 on Debian.  
**Impact:** Version disclosure allows attackers to search for known vulnerabilities 
against that specific version.  
**Remediation:** Suppress version information in Apache configuration by setting 
`ServerTokens Prod` and `ServerSignature Off` in `httpd.conf`.

---

### Finding 2 — Known CVE Against Apache 2.4.25
**Severity:** High  
**CVE:** CVE-2026-49975  
**Detail:** Apache HTTP Server versions 2.4.17 through 2.4.67 are vulnerable to 
a Denial of Service attack via malicious HTTP requests to the mod_http2 module, 
causing excessive memory allocation and server crash.  
**Impact:** An unauthenticated attacker can take the web server offline.  
**Remediation:** Upgrade Apache to version 2.4.68 or later.

---

### Finding 3 — Directory Listing Enabled on /config
**Severity:** High  
**Tool:** Gobuster  
**Command:**
```bash
gobuster dir -u http://127.0.0.1 -w /usr/share/wordlists/dirb/common.txt
```
**Result:** The `/config` directory was publicly accessible with directory listing 
enabled, exposing internal configuration files to any unauthenticated visitor.  
**Impact:** Attackers can browse and download sensitive server files without any 
authentication.  
**Remediation:** Disable directory listing in Apache by removing `Options Indexes` 
from the server configuration. Restrict access to config directories via `.htaccess` 
or firewall rules.

---

### Finding 4 — Exposed Backup Configuration File (Critical)
**Severity:** Critical  
**File:** `/config/config.inc.php.bak`  
**Detail:** A backup of the application configuration file was left in a publicly 
accessible directory. Unlike the `.php` version which executes server-side, the 
`.bak` file rendered as plain text in the browser, exposing the following:

| Parameter | Value |
|-----------|-------|
| Database host | 127.0.0.1 |
| Database name | dvwa |
| Database username | app |
| Database password | vulnerables |

**Impact:** Full database credentials exposed to any unauthenticated attacker. 
Direct database access and complete data exfiltration possible.  
**Remediation:** Never store backup files in publicly accessible directories. 
Implement a `.htaccess` rule to block access to `.bak`, `.old`, and `.dist` files. 
Audit web root regularly for leftover development files.

---

### Finding 5 — Direct Database Access via Exposed Credentials
**Severity:** Critical  
**Detail:** Using credentials from the exposed backup file, direct access to the 
MariaDB database was obtained by executing a shell inside the Docker container. 
The following query was used to dump the users table:

```sql
use dvwa;
select user, password from users;
```

**Result:** Five user accounts and their password hashes were extracted:

| Username | Hash |
|----------|------|
| admin | 5f4dcc3b5aa765d61d8327deb882cf99 |
| gordonb | e99a18c428cb38d5f260853678922e03 |
| 1337 | 8d3533d75ae2c3966d7e0d4fcc69216b |
| pablo | 0d107d09f5bbe40cade3de5c71e9e9b7 |
| smithy | 5f4dcc3b5aa765d61d8327deb882cf99 |

**Notable:** Admin and Smithy share identical hashes, confirming password reuse 
across accounts.  
**Remediation:** Never reuse passwords across accounts. Implement bcrypt or 
Argon2 for password hashing instead of MD5.

---

### Finding 6 — Weak MD5 Password Hashing
**Severity:** Critical  
**Tool:** CrackStation (online MD5 rainbow table)  
**Detail:** Password hashes were identified as unsalted MD5. The admin hash was 
submitted to CrackStation and cracked instantly:

| Hash | Plaintext |
|------|-----------|
| 5f4dcc3b5aa765d61d8327deb882cf99 | password |

**Impact:** All user passwords recoverable near-instantly using freely available 
rainbow tables. No specialist hardware required.  
**Remediation:** Replace MD5 with a modern password hashing algorithm such as 
bcrypt, scrypt, or Argon2. Enforce a minimum password complexity policy.

---

### Finding 7 — Successful Authentication as Admin
**Severity:** Critical  
**Detail:** Using the cracked plaintext password `password`, full administrative 
access to the DVWA application was obtained without ever being provided credentials.  
**Impact:** Complete application compromise. Full access to all DVWA modules and 
user data.  
**Remediation:** Enforce strong password policies. Implement account lockout after 
failed login attempts. Enable multi-factor authentication on administrative accounts.

---

## Attack Chain Summary

Nmap (version disclosure)
↓
Gobuster (directory enumeration)
↓
/config directory listing exposed
↓
config.inc.php.bak downloaded (database credentials)
↓
Docker exec → MariaDB access → users table dumped
↓
MD5 hashes cracked via CrackStation
↓
Admin login successful — full application compromise

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port scanning and service version detection |
| Gobuster | Directory and file enumeration |
| MariaDB client | Direct database access |
| CrackStation | MD5 hash cracking via rainbow tables |
| Docker | Container shell access |

---

## Lessons Learned

- Backup files left in web-accessible directories are a critical risk
- Directory listing should always be disabled in production
- MD5 is not a suitable algorithm for password storage
- Password reuse across accounts multiplies the impact of a single compromise
- Version disclosure through server headers assists attackers in targeting known CVEs

---

*This test was conducted in a controlled lab environment for educational purposes.*
