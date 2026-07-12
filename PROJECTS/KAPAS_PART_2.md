markdown# KAPAS Infrastructure — Endpoint, Peripheral, and DLP Security Controls

Group Policy-based hardening against insider threats, physical/peripheral surveillance, removable media abuse, local reconnaissance, and script/web-delivered malware.

## Overview

A dedicated GPO (`KAPAS-Infrastructure-Endpoint-Security`) was built to address twelve threats spanning data loss prevention, device/peripheral control, and endpoint malware defense — kept separate from the password/lockout/AppLocker baseline built in a prior engagement for cleaner documentation and safer rollback.

## Environment

- Windows Server 2022, Active Directory Domain Services
- Domain: `cybergonpluto.local`
- GPO: `KAPAS-Infrastructure-Endpoint-Security`
- Platform: VMware Workstation

## Threats Addressed

| # | Threat | Control | Status |
|---|---|---|---|
| 1 | USB exfiltration / ransomware payloads | Removable Disks: Deny execute + write | Configured |
| 2 | Camera espionage / spyware | Allow Use of Camera: Disabled | Configured |
| 3 | Microphone/room bugging | App Privacy: Force Deny microphone | Configured |
| 4 | Rogue Wi-Fi/network adapters | Device Installation Restrictions (Network Adapter GUID) | Configured |
| 5 | Portable apps/scripts from USB | AppLocker (prior deployment) | Already covered |
| 6 | IP leakage vs. read-only data | Same as #1, read access left allowed | Configured |
| 7 | Bluetooth side-channel extraction | Device Installation Restrictions (Bluetooth GUID) | Configured |
| 8 | Phone MTP/PTP transfer bypass | WPD Devices: Deny read + write | Configured |
| 9 | CLI/manual enumeration | Command Prompt blocked (User Config) | Configured |
| 10 | AD discovery / fileless malware | PowerShell Script Block + Module Logging | Configured |
| 11 | Proxy-disguised script execution | ASR: Block JS/VBScript downloader | Configured |
| 12 | Drive-by downloads / zipped stagers | ASR email rule + SmartScreen (Edge) | Configured — Edge-only gap documented |

## Key Design Decisions

- **Read access to removable disks was deliberately left unconfigured**, satisfying both the "block exfiltration/ransomware" (#1) and "allow read-only reference data" (#6) requirements from a single policy, by using Windows' independent read/write/execute controls rather than one blanket toggle.
- **"Also apply to matching devices that are already installed" was left unchecked** on the Device Installation Restrictions policy. Enabling it would have disabled the machine's own existing network adapter — including the one needed to stay connected to the domain — so the policy is scoped to block newly introduced devices only.
- **Two configuration mistakes were caught and corrected before being finalized**, not glossed over:
  - An incorrect policy (device *instance* ID, which targets one specific physical device) was initially used for Threats 4/7 instead of the correct device *setup class* GUID policy, which blocks an entire device category.
  - The microphone App Privacy policy was initially left at "Disabled," which — per Microsoft's own documentation — actually means no enforcement at all (user choice remains). Corrected to Enabled + Force Deny.

## Notable Technical Findings

- **SmartScreen enforcement via native Group Policy only covers Microsoft Edge.** Chrome and Firefox run independent, vendor-controlled safe-browsing systems untouched by this policy. Full coverage would require importing each browser's own ADMX templates (e.g. Google's official Chrome templates) or restricting the organization to a single approved browser via AppLocker.
- **ASR rules target behavioral patterns, not file signatures** — e.g., the rule blocking JavaScript/VBScript from launching downloaded executable content doesn't need to recognize a specific malicious script; it blocks the structural pattern itself, making it effective against novel or disguised delivery mechanisms.
- **AMSI is a hard dependency for several ASR rules**, confirmed directly in Microsoft's documentation — tying the ASR configuration back to the PowerShell logging and fileless-malware detection work done for Threat 10.
- **WPD (Windows Portable Devices) is a distinct device class from Removable Disks.** A smartphone connected via MTP/PTP does not register as a removable disk, so USB storage restrictions alone don't cover it — a separate WPD-specific policy was required.

## Repository Contents
├── KAPAS_Infrastructure_Security_Controls_Report.docx   # full report: all 12 threats,
│                                                           screenshots, mitigation logic,
│                                                           and incident response procedures
└── screenshots/                                           # GPMC configuration evidence

## Skills Demonstrated

- Group Policy administration across Removable Storage Access, Device Installation Restrictions, App Privacy, Administrative Templates, and Attack Surface Reduction
- Verifying technical claims (ASR rule GUIDs) against official Microsoft documentation rather than relying on memory
- Identifying and correcting configuration mistakes through careful validation against policy help text, rather than assuming a setting worked as intended
- Recognizing and documenting genuine control limitations (cross-browser coverage, app-level vs. system-level permission models) instead of overstating what native Group Policy can achieve
