# Windows Firewall Hardening Lab

## Overview

This project demonstrates how Windows Defender Firewall can be configured to reduce the attack surface of a Windows system through inbound and outbound filtering.

The lab focuses on creating firewall rules, restricting applications, blocking unnecessary services, and implementing a default-deny security posture.

---

## Objectives

- Block unnecessary inbound services.
- Prevent reconnaissance.
- Restrict application network access.
- Configure secure firewall defaults.
- Whitelist only essential outbound traffic.

---

## Environment

- Windows 11
- Windows Defender Firewall with Advanced Security
- VMware Workstation

---

# Tasks Completed

## 1. Block Remote Desktop (TCP 3389)

Created an inbound firewall rule that blocks all incoming Remote Desktop connections.

### Purpose

Prevent unauthorized RDP access.

---

## 2. Block ICMP Echo Requests

Configured an inbound rule to block ICMP Echo Requests.

### Purpose

Prevent the system from responding to ping scans during network reconnaissance.

---

## 3. Block Google Chrome Outbound Traffic

Created an outbound rule targeting:

```text
C:\Program Files\Google\Chrome\Application\chrome.exe
Purpose

Prevent Chrome from establishing outbound network connections.

4. Block Command Prompt Outbound Traffic

Created an outbound rule targeting:

C:\Windows\System32\cmd.exe
Purpose

Prevent the Command Prompt from initiating outbound TCP connections.

This demonstrates application-level network restrictions.

5. Restrict WinRM

Allowed inbound traffic only on:

TCP 5985
Purpose

Permit PowerShell Remoting while preventing unnecessary inbound access.

6. Configure Default-Deny Firewall Policy

Configured every firewall profile as follows:

Profile	Inbound	Outbound
Domain	Block	Block
Private	Block	Block
Public	Block	Block
7. Allow Only Web Traffic

Created an outbound rule allowing:

TCP 80
TCP 443

All other outbound traffic remained blocked unless explicitly allowed.

Skills Demonstrated
Windows Firewall Administration
Host Hardening
Network Security
Principle of Least Privilege
Application Control
Security Concepts

This lab demonstrates several important defensive security principles:

Default Deny
Least Privilege
Attack Surface Reduction
Application Whitelisting
Network Segmentation

Lessons Learned

Windows Defender Firewall provides granular control over inbound and outbound network traffic. Adopting a default-deny approach significantly reduces a host's exposure by requiring administrators to explicitly allow only the services and applications necessary for business operations.
