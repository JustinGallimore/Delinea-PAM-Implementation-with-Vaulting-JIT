<h2><a href="https://www.loom.com/share/bbca1ec3803e4b6599bd086ad354bc5b">VIDEO OVERVIEW OF THIS LAB</a></h2>
# Delinea PAM Implementation with Vaulting + JIT

![Windows Server](https://img.shields.io/badge/Windows_Server_2022-0078D4?style=for-the-badge&logo=windows&logoColor=white)
![VMware](https://img.shields.io/badge/VMware_Workstation_16-607078?style=for-the-badge&logo=vmware&logoColor=white)
![PowerShell](https://img.shields.io/badge/PowerShell-5391FE?style=for-the-badge&logo=powershell&logoColor=white)
![Active Directory](https://img.shields.io/badge/Active_Directory-0078D4?style=for-the-badge&logo=microsoft&logoColor=white)
![Delinea](https://img.shields.io/badge/Delinea_Secret_Server-FF6600?style=for-the-badge&logoColor=white)

---

## Overview

This project is a full Privileged Access Management lab built from scratch in a VMware Workstation home lab environment. It covers every core PAM discipline that enterprises pay engineers to manage in production: credential vaulting, privileged session launch through the vault, and a Just-In-Time access workflow with full audit logging.

Every phase includes real troubleshooting documentation. Nothing was hidden, cleaned up, or presented as smoother than it actually was. The errors, the root cause analysis, and the architectural decisions made to resolve them are all here with screenshots.

---

## Why This Project Exists

PAM is one of the highest-value skill sets in IAM and one of the hardest to demonstrate without enterprise access. This lab was built to close that gap. The goal was to build something that reads and functions at the level of a working engineer, not a student following a guide.

---

## What Was Built

| Phase | What Was Demonstrated |
|---|---|
| Phase 0 | VM resource planning, disk extension via diskpart, recovery partition removal, VMware snapshot strategy |
| Phase 1 | Delinea Secret Server installation including manual SQL Express install, IIS configuration, and prerequisite troubleshooting |
| Phase 2 | Credential vaulting across three account categories with organized folder structure matching enterprise standards |
| Phase 3 | Privileged session launch via RDP through the Secret Server launcher to a separate target VM, including four-attempt troubleshooting arc leading to loopback RDP root cause resolution |
| Phase 4 | Just-In-Time access workflow with comment-required checkout gate, business justification logging, and full audit trail |

---

## Lab Environment

| Component | Details |
|---|---|
| Host Machine | POWERSTATION-7, 64 GB DDR5 RAM |
| Hypervisor | VMware Workstation 16 Pro |
| Primary Server | Windows Server 2022 (DC01) at 192.168.50.10 |
| Target Server | Windows Server 2022 (WIN-TARGET01) at 192.168.50.20 |
| Domain | lab.local |
| PAM Platform | Delinea Secret Server Free Edition v12 |

---

## Tool Stack

- Delinea Secret Server Free Edition v12
- Windows Server 2022 Standard Evaluation
- VMware Workstation 16 Pro
- Active Directory Domain Services
- PowerShell
- IIS
- SQL Server 2025 Express
- Delinea Connection Manager

---

## A Note on Delinea vs CyberArk

CyberArk is the most recognized PAM platform in the enterprise space. The self-hosted version requires a license file that can only be obtained through a sales representative with no free trial available. Delinea Secret Server is a legitimate enterprise PAM platform with a free edition that supports up to 10 users and 250 secrets. Every concept demonstrated here maps directly to CyberArk PAM. The platform is different but the skills are the same.

---

## A Note on Free Edition Constraints

Several features in this lab hit the limits of the free edition including session recording playback, Secret Policy creation with checkout intervals, and multi-user access request workflows. Every constraint was documented transparently with screenshots rather than hidden. The architecture behind each of these features is fully understood and demonstrated to the extent the free edition allows.

---

## Repository Structure

```
Delinea-PAM-Implementation-with-Vaulting-JIT/
    Screenshots/          61 portfolio and troubleshooting screenshots
    README.md             This file
    WALKTHROUGH.md        Full step by step build with embedded screenshots
    TROUBLESHOOTING.md    Dedicated troubleshooting reference document
```

---

## Documentation

- [Full Walkthrough](WALKTHROUGH.md) - Complete step by step build documentation with screenshots
- [Troubleshooting Reference](TROUBLESHOOTING.md) - All errors, root causes, and resolutions in one place

---

*Built by Justin Gallimore | [github.com/JustinGallimore](https://github.com/JustinGallimore)*
