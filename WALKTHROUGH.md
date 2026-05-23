# Delinea PAM Implementation with Vaulting + JIT
### A hands-on Privileged Access Management lab built from scratch on Windows Server 2022

---

## Table of Contents

- [What This Project Is About](#what-this-project-is-about)
- [Why Delinea Instead of CyberArk](#why-delinea-instead-of-cyberark)
- [Lab Environment](#lab-environment)
- [Phase 0: Lab Preparation](#phase-0-lab-preparation)
- [Phase 1: Installing Delinea Secret Server](#phase-1-installing-delinea-secret-server)
- [Phase 2: Credential Vaulting](#phase-2-credential-vaulting)
- [Phase 3: Session Recording](#phase-3-session-recording)
- [Phase 4: Just-In-Time Access](#phase-4-just-in-time-access)
- [License Constraints Documented](#license-constraints-documented)
- [What This Demonstrates to a Hiring Manager](#what-this-demonstrates-to-a-hiring-manager)

---

## What This Project Is About

Think of your company like a building. Most employees use a regular key card to get through the front door. But some rooms need a special key. The server room. The database. The domain controller. Not just anyone should have that key, and when someone does use it, there should be a record of who used it, when, and why.

That is what Privileged Access Management is. It controls who has access to the most powerful accounts in a company. Domain administrator accounts that can change anything on the network. Service accounts that run automated processes. Local admin accounts on individual servers. These are the master keys of IT. If a bad actor gets one, it is game over.

This project builds a full PAM environment from scratch in a home lab to demonstrate exactly the skills companies pay IAM engineers to manage in production.

---

## Why Delinea Instead of CyberArk

CyberArk is the most recognized name in PAM and appears on nearly every enterprise job description. However the self-hosted version requires a license file that can only be obtained through a sales representative. There is no free trial, no direct download, and no way to get hands on it without a formal procurement process.

Delinea Secret Server is a legitimate enterprise PAM platform used by real companies. The free edition supports up to 10 users and 250 secrets, includes session recording architecture, just-in-time access workflows, and credential vaulting, and it never expires. Every concept demonstrated in this project maps directly to what CyberArk does. The tooling is different but the skills transfer completely.

This decision is documented here because transparency is part of doing the work the right way.

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

## Phase 0: Lab Preparation

Before anything could be installed, DC01 needed more resources. PAM software runs IIS, SQL, and multiple background services simultaneously. The original VM had 4 GB of RAM and 60 GB of disk which was not enough.

RAM was increased from 4 GB to 8 GB. Disk was expanded from 60 GB to 80 GB. But expanding the disk at the hypervisor level does not automatically give Windows more space. A 524 MB Recovery Partition was sitting between the C drive and the newly unallocated space, blocking the extension. Diskpart was used to delete the recovery partition and extend C drive to the full 80 GB.

A VMware snapshot named `Pre_Delinea_Install_Clean_DC01` was taken before any changes as a rollback point.

---

**DC01 disk extended to 80 GB before installation begins.**

![00](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/00_DC01_Disk_Extended_80GB_Pre_Install.png)

---

## Phase 1: Installing Delinea Secret Server

Getting the installer required creating a Delinea support portal account. The trial form on the main website rejected personal email addresses. The support portal at support.delinea.com accepted a Gmail address and the direct download link was available after login.

SQL Server Express was a dependency the installer tried to download automatically but DC01 has no internet access by design. SQL Express was downloaded on the host machine, transferred into the VM, and installed manually. After that the installer was restarted and connected to the existing SQL instance.

IIS also had to be installed. The installer detected it was missing and the Fix Issues button handled the installation. After IIS installed the installer had to be fully closed and reopened because of how it loads the Microsoft.Web.Administration assembly. The HTTPS binding showed a warning because the certificate is self-signed which is acceptable in a lab environment.

After installation completed Secret Server was accessible at `https://dc01.lab.local/SecretServer`.

---

**Delinea installer loaded on DC01 and ready to begin.**

![01](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/01_Delinea_Installer_On_DC01_Ready.png)

---

**Installer welcome screen showing version 12.**

![02](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/02_Delinea_Installer_Welcome_Screen_v12.png)

---

**SQL Express selected as the database option before the internet access issue was discovered.**

![03](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/03_Delinea_SQL_Express_Selected.png)

---

**Prerequisites showing failed state before Fix Issues was run. IIS and WCF components were missing.**

![04](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/04_Delinea_Prerequisites_Failed_Before_Fix.png)

---

**After running Fix Issues all dependencies installed successfully.**

![05](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/05_Delinea_Dependencies_Installed_Success.png)

---

**HTTPS binding warning due to self-signed certificate. IIS restart was required before the installer could continue.**

![06](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/06_Delinea_HTTPS_Binding_Error_IIS_Restart_Required.png)

---

**Prerequisites passing and installer ready to proceed.**

![07](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/07_Delinea_Prerequisites_Ready_To_Install.png)

---

**SQL Express could not download automatically because DC01 has no internet access. Manual installation was required.**

![08](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/08_Delinea_SQL_Express_No_Internet_Manual_Install_Required.png)

---

**SQL Server 2025 Express installed successfully on DC01 after manual transfer from the host machine.**

![09](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/09_SQL_Express_2025_Installed_Successfully.png)

---

**All prerequisites passing after manual SQL Express installation and IIS fix.**

![10](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/10_Delinea_Prerequisites_All_Passed_Ready.png)

---

**Database connection configured pointing to the local SQL Express instance.**

![11](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/11_Delinea_DB_Connection_Configured.png)

---

**Service account validated successfully using the LAB Administrator credential.**

![12](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/12_Delinea_Service_Account_Validated.png)

---

**Installation review screen confirming all settings before the final install.**

![13](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/13_Delinea_Installation_Review_Ready_To_Install.png)

---

**Secret Server installed successfully.**

![14](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/14_Delinea_Secret_Server_Installed_Successfully.png)

---

**Secret Server login page live and accessible at https://dc01.lab.local/SecretServer.**

![15](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/15_Delinea_Secret_Server_Login_Page_Live.png)

---

**Secret Server dashboard on first login confirming the environment is fully operational.**

![16](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/16_Delinea_Secret_Server_Dashboard_First_Login.png)

---

## Phase 2: Credential Vaulting

Vaulting credentials means storing them inside a secured audited system so that no human ever needs to know what the actual password is. When someone needs to use a privileged account they check it out from the vault, use it, and check it back in. Every access is logged.

Three credential categories were vaulted to represent the most common privileged account types found in enterprise environments.

**Folder Structure**
- Domain Admin Accounts
- Local Admin Accounts
- Service Accounts

**Secrets Created**
- `LAB Domain Administrator` in Domain Admin Accounts using the Active Directory Account template
- `SVC_Backup Service Account` in Service Accounts using the Windows Account template
- `Local Admin DC01` in Local Admin Accounts using the Windows Account template

---

**Three root level folders created matching enterprise vault organization standards.**

![17](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/17_Delinea_Three_Folders_Created.png)

---

**Domain Admin secret form ready before creation.**

![18](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/18_Domain_Admin_Secret_Ready_To_Create.png)

---

**Domain Admin secret created and stored in the vault with a vault-generated password.**

![19](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/19_Domain_Admin_Secret_Created_In_Vault.png)

---

**Service Account secret form ready before creation.**

![20](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/20_Service_Account_Secret_Ready_To_Create.png)

---

**Service Account secret created and stored in the vault.**

![21](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/21_Service_Account_Secret_Created_In_Vault.png)

---

**Local Admin secret form ready before creation.**

![22](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/22_Local_Admin_Secret_Ready_To_Create.png)

---

**Local Admin secret created and stored in the vault.**

![23](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/23_Local_Admin_Secret_Created_In_Vault.png)

---

**All three secrets vaulted across all three categories. Phase 2 complete.**

![24](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/24_All_Three_Secrets_Vaulted_Phase2_Complete.png)

---

## Phase 3: Session Recording

Session recording captures exactly what happens when a privileged account is used. Every keystroke, every command, every file accessed during a privileged session gets recorded and tied to the identity of whoever checked out the credential. This is one of the most valuable capabilities in PAM from a compliance and forensics standpoint.

### The Troubleshooting Story

This phase did not go smoothly and that is the point. Real IAM work is troubleshooting.

**Attempt 1** failed because the Delinea Connection Manager protocol handler was not installed on DC01. The 64-bit MSI was downloaded from the error dialog and installed.

**Attempt 2** failed with Error Code 516 because RDP was not enabled on DC01. PowerShell was used to enable it.

**Attempt 3** still failed with Error Code 516 because the vault-generated password in the secret did not match the actual Administrator password on DC01. The secret was updated manually.

**Attempt 4** failed again. The real root cause was identified: Secret Server was installed on DC01 and Windows blocks loopback RDP connections at the OS level by design. This is not a configuration error. It is architecture.

The solution was adding a second VM. Secret Server on DC01 needs a separate target machine to RDP into. This is the correct enterprise architecture anyway. The PAM vault server should never be the same machine as the systems being managed.

---

**Session Monitoring empty before any recordings. This is the before state.**

![25](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/25_Session_Monitoring_Empty_Before_Recording.png)

---

**Remote Desktop launcher configured to route sessions through the Secret Server client for recording.**

![26](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/26_Remote_Desktop_Launcher_SS_Client_Enabled.png)

---

**Attempt 1: Protocol handler not installed. Downloaded the 64-bit MSI from the error dialog.**

![27](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/27_Troubleshoot_Protocol_Handler_Not_Installed.png)

---

**Attempt 2: Error Code 516 because RDP was not enabled on DC01. Enabled via PowerShell.**

![28](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/28_Troubleshoot_RDP_Not_Enabled_Error_516.png)

---

**Attempt 3: Password mismatch identified between the vault-generated secret and the actual machine password.**

![29](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/29_Troubleshoot_Password_Mismatch_Identified.png)

---

**Attempt 4: Loopback RDP root cause confirmed. Windows blocks a machine from RDP-ing into itself at the OS level.**

![30](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/30_Troubleshoot_Loopback_RDP_Root_Cause_Confirmed.png)

---

### Building WIN-TARGET01

The solution was building a second VM as a dedicated RDP target. Secret Server on DC01 would then make a legitimate cross-machine RDP connection to WIN-TARGET01.

---

**New VM Wizard in VMware with Custom Advanced selected. This is the starting point for WIN-TARGET01.**

![31](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/31_WIN-TARGET01_VMware_New_VM_Wizard_Custom_Selected.png)

---

**WIN-TARGET01 VM configured with 2 GB RAM, 40 GB NVMe disk, and Bridged networking before power on.**

![32](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/32_WIN-TARGET01_VM_Hardware_Configuration_Summary.png)

---

**Windows Server 2022 Standard Evaluation installed and at the desktop on WIN-TARGET01.**

![33](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/33_WIN-TARGET01_Windows_Server_2022_Install_Complete_Desktop.png)

---

**Static IP configured at 192.168.50.20 with DNS pointing to DC01 at 192.168.50.10.**

![34](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/34_WIN-TARGET01_Static_IP_Configured_192.168.50.20.png)

---

**WIN-TARGET01 login screen after domain join restart showing Other User option confirming domain membership.**

![35](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/35_WIN-TARGET01_Post_Domain_Join_Login_Screen.png)

---

**Domain membership confirmed. WMI query returns lab.local and the prompt shows Administrator.LAB.**

![36](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/36_WIN-TARGET01_Domain_Membership_Confirmed_Lab_Local.png)

---

**RDP enabled on WIN-TARGET01 via PowerShell.**

![37](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/37_WIN-TARGET01_RDP_Enabled_Via_PowerShell.png)

---

**RDP firewall rules confirmed enabled via Get-NetFirewallRule.**

![38](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/38_WIN-TARGET01_RDP_Firewall_Rules_Confirmed_Enabled.png)

---

**VMware snapshot taken before touching Secret Server. Clean rollback point established.**

![39](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/39_WIN-TARGET01_Snapshot_Taken_Clean_State.png)

---

**Connectivity verified from DC01 to WIN-TARGET01 on port 3389 before assuming the tool would work. TcpTestSucceeded: True.**

![40](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/40_DC01_Test_NetConnection_WIN-TARGET01_Port_3389_Success.png)

---

### Running the Session Through Secret Server

---

**New secret targeting WIN-TARGET01 at 192.168.50.20 ready to create in the Local Admin Accounts folder.**

![41](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/41_Secret_Local_Admin_WIN-TARGET01_Ready_To_Create.png)

---

**Secret created and stored in the vault. RDP launcher visible and Active sessions showing 0 before launch.**

![42](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/42_Secret_Local_Admin_WIN-TARGET01_Created_In_Vault.png)

---

**RDP certificate warning from WIN-TARGET01 confirming the launcher made a real cross-machine connection. Certificate name shows WIN-TARGET01.lab.local.**

![43](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/43_Secret_Launcher_RDP_Certificate_Warning_WIN-TARGET01.png)

---

**Live RDP session connected to WIN-TARGET01 through Secret Server. Title bar confirms 192.168.50.20.**

![44](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/44_Active_RDP_Session_Connected_To_WIN-TARGET01.png)

---

**Commands run inside the recorded session. whoami returns win-target01\administrator. hostname returns WIN-TARGET01. ipconfig confirms 192.168.50.20. The vaulted credential authenticated against a separate machine without anyone seeing the password.**

![45](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/45_Active_RDP_Session_Commands_Run_Recorded.png)

---

**Session Monitoring configuration is gated behind the paid license on the free edition. The launcher and cross-machine RDP architecture are fully proven. Full session recording playback requires Secret Server Professional Edition or higher.**

![46](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/46_Session_Monitoring_Page_License_Gate_Noted.png)

---

## Phase 4: Just-In-Time Access

Just-In-Time access means someone has to request elevated access, provide a justification, and that access is controlled and audited. This is Zero Trust access control in practice. Right access, right time, fully logged.

The free edition user limit of 1 account blocked creating a second user for a two-party request and approval flow. This is documented transparently below. The JIT controls demonstrated are the comment-required checkout gate, justified access logging, and the full audit trail.

---

**Secret Server dashboard before JIT configuration. Approvals widget shows no pending approvals. This is the clean before state.**

![47](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/47_Secret_Server_Dashboard_Before_JIT_Configuration.png)

---

**JIT test user form filled out correctly before hitting the license limit.**

![48](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/48_JIT_Test_User_Form_Ready_To_Create.png)

---

**Free edition user license limit reached at 1 user. Second user creation blocked. Documented transparently.**

![49](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/49_JIT_User_Creation_License_Limit_Free_Edition.png)

---

**Comment required gate active. Secret Server immediately enforces the justification requirement before any access is granted. Nobody gets in without providing a reason.**

![50](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/50_Secret_Checkout_Comment_Required_Gate_Active.png)

---

**Business justification typed before submitting. Ticket reference included matching real enterprise change control standards.**

![51](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/51_JIT_Access_Request_Comment_Submitted.png)

---

**Access granted after comment submission. Secret detail view now accessible with all fields visible.**

![52](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/52_JIT_Access_Granted_Secret_Now_Accessible.png)

---

**Audit log showing the full lifecycle of this secret. Top entry shows the justified access event with the maintenance comment. Below that shows the RDP Launch event from the session recording phase pointing to WIN-TARGET01. Bottom shows the original Create event. Every action is logged, timestamped, and tied to an identity.**

![53](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/53_JIT_Audit_Log_Access_Comment_Recorded.png)

---

**Comment gate enforcing on every single access attempt, not just the first. Consistent access control across all sessions.**

![54](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/54_JIT_Comment_Gate_Enforcing_On_Every_Access.png)

---

**Comment typed with a second business justification before submitting.**

![55](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/55_JIT_Comment_Typed_Before_Submit.png)

---

**Security tab confirming Require comment is set to Yes on the secret.**

![56](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/56_Secret_Security_Tab_Require_Comment_Confirmed_Yes.png)

---

**Secret Policy page showing the license warning for the free edition.**

![57](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/57_Secret_Policy_License_Warning_Free_Edition.png)

---

**JIT Access Policy configured correctly with name and description showing the intended 4-hour checkout window architecture. Save blocked by free edition license.**

![58](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/58_JIT_Secret_Policy_License_Blocked_Free_Edition.png)

---

**Advanced information section on the secret showing the Secret policy field where the JIT policy would be applied in a licensed environment.**

![59](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/59_Secret_Advanced_Info_Policy_Field_Visible.png)

---

**Settings tab confirming checkout interval configuration is not available in the free edition. Documented transparently.**

![60](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/60_Secret_Settings_Tab_Checkout_Not_Available_Free_Edition.png)

---

## License Constraints Documented

Throughout this lab the free edition of Delinea Secret Server imposed the following limits. Each was documented transparently with screenshots rather than hidden or worked around.

| Feature | Free Edition | Professional Edition |
|---|---|---|
| User accounts | 1 maximum | Unlimited |
| Session recording playback | Not available | Available |
| Secret Policy creation | Blocked | Available |
| Checkout interval configuration | Not available | Available |
| Two-party access request workflow | Blocked by user limit | Available |

These constraints do not diminish what was demonstrated. The architecture, the troubleshooting process, the credential vaulting, the cross-machine RDP launch through the vault, and the justified access audit trail are all fully proven. In a production environment with a licensed edition every one of these features would be configured and enforced exactly as designed in this lab.

---

## What This Demonstrates to a Hiring Manager

Every phase maps to something a PAM engineer does on the job.

**VM prep and resource planning** maps to infrastructure readiness before a PAM deployment.

**Dependency troubleshooting during installation** maps to production deployment experience where things never go cleanly.

**Credential vaulting with organized folder structure** maps to day-one PAM hygiene work that separates engineers who understand governance from those who just follow guides.

**Session recording with cross-machine architecture** maps to understanding why PAM vault servers must be isolated from the systems they manage. This is not a config setting. It is an architectural principle.

**JIT access with justified checkout and audit logging** maps to zero trust access principles that every mature enterprise is actively building toward.

**The troubleshooting narrative in Phase 3** is the strongest part of this portfolio. Four failed attempts, each diagnosed correctly, leading to an architectural root cause that required building a second VM to resolve. That is not a student following a tutorial. That is an engineer thinking through a problem.

---

*Built by Justin Gallimore | github.com/JustinGallimore*
