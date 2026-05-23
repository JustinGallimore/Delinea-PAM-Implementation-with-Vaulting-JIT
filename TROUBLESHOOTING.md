# Troubleshooting Reference

### Delinea PAM Implementation with Vaulting + JIT

This document captures every error, root cause, and resolution encountered during this lab build. It exists as a standalone reference because troubleshooting is where real PAM engineering happens. Clean installs that work on the first try do not build diagnostic skills. These did not.

---

## Table of Contents

- [Issue 01: Disk Extension Blocked by Recovery Partition](#issue-01-disk-extension-blocked-by-recovery-partition)
- [Issue 02: SQL Express Failed to Download During Installation](#issue-02-sql-express-failed-to-download-during-installation)
- [Issue 03: IIS Not Installed Before Delinea Installer Ran](#issue-03-iis-not-installed-before-delinea-installer-ran)
- [Issue 04: Installer Failed to Recognize IIS After Fix Issues](#issue-04-installer-failed-to-recognize-iis-after-fix-issues)
- [Issue 05: HTTPS Binding Warning on Self-Signed Certificate](#issue-05-https-binding-warning-on-self-signed-certificate)
- [Issue 06: HTTP Error 503 Service Unavailable After Idle](#issue-06-http-error-503-service-unavailable-after-idle)
- [Issue 07: Protocol Handler Not Installed on First Launch Attempt](#issue-07-protocol-handler-not-installed-on-first-launch-attempt)
- [Issue 08: Error Code 516 RDP Not Enabled](#issue-08-error-code-516-rdp-not-enabled)
- [Issue 09: Error Code 516 Password Mismatch Between Vault and Machine](#issue-09-error-code-516-password-mismatch-between-vault-and-machine)
- [Issue 10: Error Code 516 Loopback RDP Root Cause](#issue-10-error-code-516-loopback-rdp-root-cause)
- [Issue 11: WIN-TARGET01 Interface Name Verification Required](#issue-11-win-target01-interface-name-verification-required)
- [Issue 12: RDP Certificate Warning on First Launcher Connection](#issue-12-rdp-certificate-warning-on-first-launcher-connection)
- [Issue 13: Session Recording Configuration Access Denied](#issue-13-session-recording-configuration-access-denied)
- [Issue 14: Free Edition User License Limit Blocks JIT Test User](#issue-14-free-edition-user-license-limit-blocks-jit-test-user)
- [Issue 15: Secret Policy Save Blocked by License Gate](#issue-15-secret-policy-save-blocked-by-license-gate)
- [Issue 16: Checkout Interval Not Available in Free Edition](#issue-16-checkout-interval-not-available-in-free-edition)

---

## Issue 01: Disk Extension Blocked by Recovery Partition

**Phase:** Phase 0

**What Happened**

The DC01 VM disk was expanded from 60 GB to 80 GB inside VMware Workstation settings. After the expansion Windows did not recognize the additional space. Disk Management showed the 20 GB of unallocated space but the Extend Volume option was grayed out on the C drive.

**Root Cause**

A 524 MB Recovery Partition was sitting between the C drive partition and the newly unallocated space. Windows will not extend a volume across a partition that sits between the drive and the free space. The recovery partition had to be removed first.

**Resolution**

Diskpart was used to delete the recovery partition and extend C drive to the full 80 GB.

```
diskpart
list disk
select disk 0
list partition
select partition 4
delete partition override
select partition 3
extend
exit
```

**Screenshot**

![00](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/00_DC01_Disk_Extended_80GB_Pre_Install.png)

**Lesson**

Expanding a VM disk at the hypervisor level only gives Windows the raw space. Partition layout inside the OS controls whether that space is usable. Always check partition layout before assuming an extension will work cleanly.

---

## Issue 02: SQL Express Failed to Download During Installation

**Phase:** Phase 1

**What Happened**

The Delinea installer attempted to download SQL Server Express automatically during the installation prerequisites check. The download failed and the SQL Express prerequisite showed as Failed.

**Root Cause**

DC01 has no internet access by design. The installer tried to reach Microsoft download servers and could not make the connection.

**Resolution**

SQL Server 2025 Express was downloaded manually on the host machine, transferred into the DC01 VM using VMware drag and drop, and installed manually using the Basic installation type. After installation completed the Delinea installer was restarted and the Connect to an existing SQL Server option was selected, pointing to localhost\SQLEXPRESS.

**Screenshot**

![08](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/08_Delinea_SQL_Express_No_Internet_Manual_Install_Required.png)

**Lesson**

Any installer that pulls dependencies from the internet will fail on an air-gapped or network-isolated VM. Always pre-stage dependencies before running enterprise software installers in isolated lab environments.

---

## Issue 03: IIS Not Installed Before Delinea Installer Ran

**Phase:** Phase 1

**What Happened**

The Delinea prerequisites check showed IIS Installed as Failed. Secret Server requires IIS to host the web application and the Windows Server 2022 base installation does not include IIS by default.

**Root Cause**

IIS is not installed by default on Windows Server 2022. The Delinea installer detected the missing role and flagged it.

**Resolution**

The Fix Issues button inside the Delinea installer automatically installed IIS along with the required IIS components and .NET WCF Activation. After Fix Issues completed all prerequisites moved from Failed to Passed.

**Screenshot**

![04](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/04_Delinea_Prerequisites_Failed_Before_Fix.png)

**Lesson**

Enterprise web applications almost always require IIS to be pre-staged on Windows Server. The Delinea installer handles this automatically through Fix Issues but in production you would pre-install IIS as part of the server build standard before any application installer runs.

---

## Issue 04: Installer Failed to Recognize IIS After Fix Issues

**Phase:** Phase 1

**What Happened**

After Fix Issues ran and IIS installed successfully the Delinea installer still showed IIS prerequisites as failed even though IIS was confirmed installed and running.

**Root Cause**

The Delinea installer loads the Microsoft.Web.Administration assembly at startup. When IIS is installed mid-session the assembly is already loaded in memory from the pre-IIS state. The installer cannot detect the new IIS installation without reloading the assembly, which requires a full restart of the installer process.

**Resolution**

The installer was fully closed and reopened. On relaunch it loaded the Microsoft.Web.Administration assembly fresh, detected IIS correctly, and all prerequisites passed.

**Lesson**

When an installer installs a dependency mid-session always close and reopen the installer completely before continuing. In-memory state does not update automatically when new components are added to the system during the same session.

---

## Issue 05: HTTPS Binding Warning on Self-Signed Certificate

**Phase:** Phase 1

**What Happened**

The HTTPS Binding prerequisite showed as a Warning rather than Passed after IIS was installed. The installer flagged that the certificate was not from a trusted certifying authority.

**Root Cause**

Secret Server requires HTTPS. When IIS was installed the only available certificate was the default self-signed certificate generated by IIS. Self-signed certificates are not trusted by default because they are not issued by a recognized certificate authority.

**Resolution**

The Warning was accepted for the lab environment. In production a certificate from an internal CA or a public CA would be bound to the IIS site. For a home lab the self-signed certificate is sufficient and the browser warning is expected behavior.

**Screenshot**

![06](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/06_Delinea_HTTPS_Binding_Error_IIS_Restart_Required.png)

**Lesson**

Self-signed certificates are acceptable in lab environments but must be replaced with CA-issued certificates in production. The warning does not block functionality. It is a reminder that the trust chain is incomplete.

---

## Issue 06: HTTP Error 503 Service Unavailable After Idle

**Phase:** Multiple

**What Happened**

After leaving DC01 idle overnight and returning to the lab Secret Server was unreachable at `https://dc01.lab.local/SecretServer`. The browser returned HTTP Error 503 Service Unavailable.

**Root Cause**

IIS application pools have a default idle timeout of 20 minutes. When no requests are made to the application pool for 20 minutes IIS stops the worker process to conserve resources. The SecretServer application pool stopped itself during the idle period.

**Resolution**

IIS was restarted using the following commands in PowerShell:

```powershell
iisreset /stop
iisreset /start
```

To prevent future timeouts the idle timeout was set to zero:

```powershell
Set-ItemProperty "IIS:\AppPools\SecretServer" -Name processModel.idleTimeout -Value 00:00:00
```

**Lesson**

In production the idle timeout setting is managed through Group Policy or IIS configuration standards. In a home lab where the VM is frequently paused or left idle this setting must be explicitly set to zero or the application pool will stop itself regularly. This command may need to be rerun after every IIS reset since resets clear runtime state.

---

## Issue 07: Protocol Handler Not Installed on First Launch Attempt

**Phase:** Phase 3

**What Happened**

Clicking the RDP Launcher on a secret for the first time produced an error stating the protocol handler failed to launch. The session did not open.

**Root Cause**

The Delinea Connection Manager uses a custom protocol handler registered on the local machine to intercept RDP launch requests from the browser and route them through the Connection Manager client. This protocol handler is a separate MSI that is not installed as part of Secret Server itself.

**Resolution**

The error dialog included a download link for the protocol handler MSI. The 64-bit version was downloaded and installed on DC01. After installation the launcher worked correctly on the next attempt.

**Screenshot**

![27](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/27_Troubleshoot_Protocol_Handler_Not_Installed.png)

**Lesson**

Any machine that will be used to launch sessions through Secret Server needs the Connection Manager and protocol handler installed. In an enterprise deployment this would be pushed via GPO or SCCM to all engineer workstations as part of the PAM onboarding standard.

---

## Issue 08: Error Code 516 RDP Not Enabled

**Phase:** Phase 3

**What Happened**

After installing the protocol handler the RDP launcher attempted to connect but returned Error Code 516: Unable to connect. Ensure that remote management is enabled on the target server and the host is reachable.

**Root Cause**

RDP was not enabled on DC01. Windows Server 2022 ships with RDP disabled by default. The fDenyTSConnections registry value was set to 1 blocking all incoming RDP connections.

**Resolution**

RDP was enabled via PowerShell:

```powershell
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name fDenyTSConnections -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
```

**Screenshot**

![28](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/28_Troubleshoot_RDP_Not_Enabled_Error_516.png)

**Lesson**

Error Code 516 from the Delinea launcher is a generic connectivity error. It does not specify whether the issue is RDP disabled, a firewall rule, a credential problem, or an architectural issue. Always work through possible causes systematically rather than jumping to the most complex explanation first.

---

## Issue 09: Error Code 516 Password Mismatch Between Vault and Machine

**Phase:** Phase 3

**What Happened**

After enabling RDP the launcher still returned Error Code 516. The connection attempt was reaching DC01 but failing at authentication.

**Root Cause**

The secret in Secret Server was using a vault-generated password that did not match the actual Administrator password set on DC01. Secret Server generated a strong random password when the secret was created but the actual account on the machine still had the original password. The vault and the machine were out of sync.

**Resolution**

The secret was edited manually and the password field was updated to match the actual Administrator password on DC01. In a production environment this would be resolved by enabling Secret Server heartbeat and remote password changing so the vault manages the password rotation automatically and the two are never out of sync.

**Screenshot**

![29](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/29_Troubleshoot_Password_Mismatch_Identified.png)

**Lesson**

Vault-generated passwords only match the actual account if Secret Server performed the initial password set or if remote password changing is enabled and has run at least once. When onboarding existing accounts the current password must be manually entered first before the vault can take over rotation.

---

## Issue 10: Error Code 516 Loopback RDP Root Cause

**Phase:** Phase 3

**What Happened**

After fixing the password mismatch the launcher still returned Error Code 516 on the fourth attempt. All previous causes had been ruled out. RDP was enabled, the firewall rule was open, the credentials were correct.

**Root Cause**

Secret Server was installed on DC01. The launcher was attempting to RDP from DC01 into DC01. Windows blocks loopback RDP connections at the OS level by design. A machine cannot RDP into itself regardless of credentials or configuration. This is a Windows security restriction not a Delinea configuration error.

**Resolution**

A second VM was built. WIN-TARGET01 was created as a dedicated RDP target running Windows Server 2022, joined to lab.local, assigned a static IP at 192.168.50.20, and configured with RDP enabled. A new secret was created in Secret Server targeting WIN-TARGET01. The launcher then made a legitimate cross-machine RDP connection from DC01 to WIN-TARGET01 and connected successfully.

**Screenshot**

![30](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/30_Troubleshoot_Loopback_RDP_Root_Cause_Confirmed.png)

**Lesson**

The PAM vault server should never be the same machine as the systems being managed. This is not just a lab constraint. It is a core architectural principle. In production the Secret Server instance runs on dedicated infrastructure separate from every managed system. The loopback restriction reinforced this principle in a way that no documentation could.

---

## Issue 11: WIN-TARGET01 Interface Name Verification Required

**Phase:** Phase 3

**What Happened**

Before running the static IP configuration commands on WIN-TARGET01 there was a risk that the network interface might not be named Ethernet0. If the interface name was different the New-NetIPAddress command would fail.

**Root Cause**

VMware assigns interface names based on the virtual NIC driver. The name Ethernet0 is typical for VMware VMs but not guaranteed in every configuration.

**Resolution**

Get-NetAdapter was run first to confirm the interface name before running the IP configuration commands.

```powershell
Get-NetAdapter
```

The output confirmed the interface was named Ethernet0 and the static IP commands were run successfully.

**Lesson**

Never assume interface names. Always verify with Get-NetAdapter before running any network configuration commands. One wrong interface name silently misconfigures the wrong adapter or throws an error that is harder to diagnose than the original check would have been.

---

## Issue 12: RDP Certificate Warning on First Launcher Connection

**Phase:** Phase 3

**What Happened**

When the RDP launcher connected to WIN-TARGET01 for the first time a certificate warning dialog appeared stating the identity of the remote computer cannot be verified. The certificate name showed WIN-TARGET01.lab.local and the error indicated the certificate was not from a trusted certifying authority.

**Root Cause**

WIN-TARGET01 uses the default self-signed certificate generated by Windows for RDP connections. This certificate is not issued by a CA that the connecting machine trusts so the RDP client warns before proceeding.

**Resolution**

This is expected behavior in a lab environment. The warning was accepted by clicking Yes. The checkbox to not ask again for this computer was checked to suppress the warning on future connections to this specific host.

In production this would be resolved by deploying RDP certificates from the internal CA via Group Policy, which pushes trusted certificates to all domain-joined machines automatically.

**Screenshot**

![43](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/43_Secret_Launcher_RDP_Certificate_Warning_WIN-TARGET01.png)

**Lesson**

RDP certificate warnings in a domain environment are a signal that certificate auto-enrollment via GPO is not configured. In a mature enterprise every domain-joined machine receives a computer certificate from the internal CA and RDP warnings do not appear for internal connections.

---

## Issue 13: Session Recording Configuration Access Denied

**Phase:** Phase 3

**What Happened**

Attempting to navigate to the session recording configuration settings in Secret Server returned an Access Denied page with a No Permission message regardless of which URL path was used.

**Root Cause**

Session recording configuration is gated behind the Secret Server Professional Edition license. The free edition does not grant access to this configuration page even for the Administrator account. The access denial is a license enforcement mechanism not a permissions issue with the user account.

The Administrator role was confirmed correctly assigned to the account. The issue was confirmed as a license gate after exhausting all navigation paths.

**Resolution**

This constraint was documented transparently. The session recording architecture was proven through the cross-machine RDP launcher connection. The recording configuration and playback features require a licensed edition.

**Screenshot**

![46](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/46_Session_Monitoring_Page_License_Gate_Noted.png)

**Lesson**

When an access denied error persists despite confirmed correct role assignments the next diagnostic step is checking whether the feature itself is license-gated rather than permission-gated. These are two different problems with different solutions. Role assignments fix permission issues. A license fixes license issues.

---

## Issue 14: Free Edition User License Limit Blocks JIT Test User

**Phase:** Phase 4

**What Happened**

Attempting to create a second user account in Secret Server to simulate a two-party JIT access request and approval workflow produced an error stating no licenses are currently available for new users.

**Root Cause**

The Delinea Secret Server free edition is limited to 1 enabled user account. The admin account created during installation consumed the only available license seat. A second user cannot be created without upgrading to a paid edition.

**Resolution**

The JIT workflow was demonstrated using the admin account with the comment-required checkout gate enforcing justified access on every attempt. The audit trail captured every access event with the business justification attached. The architecture for a two-party workflow was documented and the policy configuration was built to the point the license allowed.

**Screenshot**

![49](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/49_JIT_User_Creation_License_Limit_Free_Edition.png)

**Lesson**

Free editions of enterprise PAM platforms are designed for evaluation not full workflow simulation. Understanding exactly where the license limits fall and being able to articulate what a production deployment would look like beyond those limits is part of demonstrating real platform knowledge.

---

## Issue 15: Secret Policy Save Blocked by License Gate

**Phase:** Phase 4

**What Happened**

A Secret Policy named JIT_Access_Policy_4HR was configured with the correct name, description, and enabled state. Clicking Save produced a license error stating Secret policy requires an additional license: Secret Server Free or higher.

**Root Cause**

Secret Policy creation and management is a licensed feature in Delinea Secret Server. The free edition displays the Secret Policy interface and allows navigation to the creation form but blocks the save operation with a license enforcement error.

**Resolution**

The policy configuration was documented with a screenshot showing the correctly filled form and the license gate error. The intended architecture of a 4-hour checkout window enforced through the policy was documented in the WALKTHROUGH and README.

**Screenshot**

![58](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/58_JIT_Secret_Policy_License_Blocked_Free_Edition.png)

**Lesson**

Showing that you built the configuration correctly and understood where the license wall was is more valuable than pretending the feature worked. Any experienced PAM engineer reviewing this portfolio will know exactly what the policy would have done and will see that the architecture was understood completely.

---

## Issue 16: Checkout Interval Not Available in Free Edition

**Phase:** Phase 4

**What Happened**

After the Secret Policy was blocked the Settings tab on the secret was reviewed for a direct checkout interval configuration option. No Check Out section was present on the Settings tab. The sections available were Email Notifications, RDP Launcher, Expiration, and Display Icon Settings only.

**Root Cause**

The checkout interval and require checkout features are tied to Secret Policy enforcement in Delinea Secret Server. Because Secret Policy is a licensed feature the checkout interval configuration that would normally appear as a secret-level setting is also unavailable in the free edition.

**Resolution**

This constraint was documented. The comment-required gate was used as the available JIT control within the free edition. Every access attempt required a typed business justification logged to the audit trail with a timestamp and identity.

**Screenshot**

![60](https://raw.githubusercontent.com/JustinGallimore/Delinea-PAM-Implementation-with-Vaulting-JIT/main/Screenshots/60_Secret_Settings_Tab_Checkout_Not_Available_Free_Edition.png)

**Lesson**

In a licensed production environment the checkout interval would be set through the Secret Policy applied to the secret. The policy would enforce a maximum checkout duration after which Secret Server automatically revokes access and rotates the password. This is the automatic revocation piece of a complete JIT workflow. The concept is fully understood. The execution was limited by the edition.

---

## Summary

| Issue | Category | Resolution |
|---|---|---|
| Recovery partition blocking disk extension | Infrastructure | Diskpart delete partition and extend |
| SQL Express no internet access | Installation | Manual download and transfer |
| IIS not installed | Installation | Fix Issues button in installer |
| IIS not recognized after install | Installation | Close and reopen installer |
| HTTPS self-signed certificate warning | Configuration | Accepted for lab, CA cert required in production |
| HTTP 503 after idle | IIS | iisreset and idle timeout set to zero |
| Protocol handler not installed | Session Launch | Downloaded and installed 64-bit MSI |
| Error 516 RDP not enabled | Session Launch | Enabled via PowerShell registry and firewall rule |
| Error 516 password mismatch | Session Launch | Updated secret to match actual machine password |
| Error 516 loopback RDP | Architecture | Built WIN-TARGET01 as dedicated target VM |
| Interface name assumption | Networking | Verified with Get-NetAdapter before configuration |
| RDP certificate warning | Security | Accepted for lab, CA cert deployment via GPO in production |
| Session recording access denied | License | Documented transparently, architecture proven via launcher |
| User license limit | License | Documented transparently, JIT demonstrated via comment gate |
| Secret Policy save blocked | License | Documented transparently, policy architecture documented |
| Checkout interval unavailable | License | Documented transparently, concept fully understood |

---

*Built by Justin Gallimore | [github.com/JustinGallimore](https://github.com/JustinGallimore)*
