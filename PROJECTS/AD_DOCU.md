# Active Directory Lab Documentation

## Overview

This project documents the creation of a Windows Server 2022 Active Directory environment using VMware Workstation. The lab focused on designing an Organizational Unit (OU) structure, creating users, and managing domain-wide password policies through Group Policy.

> **Environment**
>
> - Windows Server 2022
> - VMware Workstation
> - Active Directory Domain Services (AD DS)
> - Domain: `cybergonpluto.local`

---

## Objectives

- Install and configure Active Directory.
- Design an Organizational Unit hierarchy.
- Create department-specific users.
- Apply Group Policy configurations.
- Understand domain-level password management.

---

## Organizational Structure
CBNX
│
├── BANKING
│ ├── Retail
│ ├── Finance
│ ├── Loans
│ └── Compliance
│
├── HEALTH
│ ├── Pharmacy
│ ├── Nursing
│ ├── Records
│ └── Administration
│
└── EDUCATION
├── Admissions
├── ICT
├── Faculty
└── Finance


Each department contains two unique user accounts.

**Total**

- 3 Sector OUs
- 12 Department OUs
- 24 Users

---

## Technologies Used

- Windows Server 2022
- Active Directory Administrative Center
- Group Policy Management Console
- VMware Workstation

---

## Configuration Steps

### 1. Created Organizational Units

- Created the parent organization (CBNX)
- Created Banking, Health and Education OUs
- Added four departmental OUs beneath each sector

---

### 2. Created User Accounts

For every department:

- Created two unique users
- Configured User Principal Names (UPNs)
- Assigned initial passwords
- Disabled "User must change password at next logon"

---

### 3. Password Policy Configuration

Using the Default Domain Policy, password complexity was disabled.

Policy Path:


Computer Configuration
→ Policies
→ Windows Settings
→ Security Settings
→ Account Policies
→ Password Policy
→ Password must meet complexity requirements


Setting:


Disabled


After running:

```powershell
gpupdate /force

the policy propagated across the domain.

Verification

The following were successfully verified:

Three sector Organizational Units
Four departments under each sector
Twenty-four unique users
Password policy applied domain-wide
Skills Demonstrated
Active Directory Administration
Organizational Unit Design
User Management
Group Policy Configuration
Windows Server Administration
