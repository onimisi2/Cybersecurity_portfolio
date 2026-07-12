# KAPAS International — Enterprise Security Controls Implementation

Group Policy-based hardening of an Active Directory domain against eleven identity and endpoint attack vectors, implemented and documented as part of a practical cybersecurity engineering assignment.

## Overview

This project simulates a real-world enterprise hardening engagement: given a list of identified threats against a fictional organization (KAPAS INTERNATIONAL), configure, test, and document the Active Directory Group Policy controls needed to mitigate each one.

Rather than treating this as a checklist exercise, every control in this repo is broken down into three parts:

1. **Implementation** — the exact Group Policy Object (GPO) path and configuration applied, with screenshot evidence
2. **Logic** — the specific mechanism of the attack, and why the configured control defeats it
3. **Incident Response** — detection indicators (mapped to Windows Event IDs), isolation steps, and recovery procedure if the control is bypassed

This structure was chosen deliberately: a control that's configured but not understood isn't a real defense, and a defense with no fallback plan isn't complete.

## Lab Environment

- **OS:** Windows Server 2022, Active Directory Domain Services
- **Domain:** `cybergonpluto.local`
- **Tooling:** Group Policy Management Console (GPMC)
- **Platform:** VMware Workstation

## Threats Addressed

| # | Threat | Primary Control | Status |
|---|---|---|---|
| 1 | Offline NTLM hash cracking / automated brute-force | Minimum password length (14 chars) + Account Lockout Policy | Configured |
| 2 | Dictionary attacks exploiting guessable patterns | Password complexity requirements | Configured |
| 3 | Identity reuse — cycling between familiar passwords | Enforce Password History (24 remembered) | Configured |
| 4 | Bypassing password history via instant revert | Minimum Password Age (1 day) | Configured |
| 5 | Brute-force / dictionary sprays on exposed accounts | Account Lockout Policy (5 attempts) | Configured |
| 6 | Directory-wide password spray sweeps | Account Lockout Policy | Partially mitigated — documented |
| 7 | Persistence via stale or leaked credentials | Maximum Password Age (42 days) | Configured |
| 8 | Domain escalation via Tier 0 admin logon to Tier 2 | Deny log on locally (Domain Admins group) | Configured, not yet linked |
| 9 | Users/exploits disabling firewalls or antivirus | Defender tamper block + Firewall enforced on all profiles | Configured |
| 10 | LLMNR / NetBIOS sniffing and poisoning | LLMNR disabled via GPO; NetBIOS documented | Configured |
| 11 | Unapproved software / untrusted executable malware | AppLocker (Executable, Installer, Script rules) | Configured |

## Key Design Decisions

**Account lockout duration was set to an aggressive 2000 minutes.** This was a deliberate trade-off, accepted because lockouts remain manually reversible by an administrator — prioritizing resistance to sustained brute-force over user convenience.

**The Tier 0 / Tier 2 logon restriction GPO was built, tested, and intentionally left unlinked.** The "Deny log on locally" right was correctly configured against the Domain Admins security group, and group membership was verified. However, the current Active Directory build-out contains only user and group objects — no dedicated Tier 2 workstation Organizational Unit exists yet. Linking the GPO prematurely (e.g. to the domain root) would have denied Domain Admins the ability to log into the Domain Controller itself, breaking the environment. This is documented as a known limitation with the correct production linkage path, rather than worked around with an unsafe shortcut.

**Tamper Protection was identified as out of scope for on-premises Group Policy — by Microsoft's own design.** Microsoft deliberately removed Tamper Protection from GPO control around 2019–2020, specifically because an attacker with GPO-level access could otherwise use it to disable the protection meant to stop them. The equivalent risk was instead mitigated through the "Turn off Microsoft Defender Antivirus" policy (locked to Disabled) and enforced firewall state across all three network profiles. In a cloud-managed environment, Tamper Protection would be enforced through Microsoft Intune or Defender for Endpoint instead.

**Password spray attacks are only partially covered by standard Account Lockout Policy**, since a spray attack tries one password against many accounts rather than many passwords against one — no single account accumulates enough failed attempts to trigger the lockout threshold. This limitation is documented explicitly rather than presented as fully solved; true directory-wide spray detection requires tooling such as Microsoft Entra ID Smart Lockout or SIEM correlation.

## Notable Technical Findings

- **AppLocker enforcement has a hard dependency on the Application Identity service.** Without it running, AppLocker rules are silently ignored regardless of how correctly they're configured. This was addressed by setting the service to Automatic startup via GPO (`Computer Configuration → Security Settings → System Services`).
- **NetBIOS has no single domain-wide GPO toggle**, unlike LLMNR. Its `NetbiosOptions` registry value is stored per network adapter, keyed by adapter GUID. Two implementation paths were evaluated: a Group Policy Preferences registry item per adapter, versus disabling NetBIOS at the DHCP scope level. The DHCP-scope approach was identified as the more practical, adapter-agnostic solution for production rollout.

## Repository Contents
