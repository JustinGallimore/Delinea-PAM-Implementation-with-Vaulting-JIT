# Lessons Learned

## Delinea-PAM-Implementation-with-Vaulting-JIT

This document covers what broke during the build, what caused it, how it was fixed, and what the production equivalent would look like. These are not hypothetical scenarios. Every issue here happened in a live lab environment and required real troubleshooting to resolve.

---

## Issue 01: SQL Express Download Failed on Air-Gapped VM

**What broke:**

The Delinea installer attempted to download SQL Server Express automatically during the prerequisites check. The download failed and the SQL Express prerequisite showed as Failed immediately.

**Root cause:**

DC01 has no internet access by design. The installer tried to reach Microsoft download servers and could not make the connection. Any installer that pulls dependencies from the internet will fail on a network-isolated VM.

**How it was fixed:**

SQL Server Express was downloaded manually on the host machine, transferred into DC01 using VMware drag and drop, and installed using the Basic installation type. After manual installation the Delinea installer was restarted and the Connect to an existing SQL Server option was selected pointing to localhost\SQLEXPRESS.

**What production looks like:**

Enterprise software deployments in production environments almost never have internet access. All dependencies are pre-staged in an internal software repository before any installer runs. The deployment runbook specifies every prerequisite and the internal path to obtain it. Nothing is downloaded from the internet during an installation.

---

## Issue 02: IIS Not Recognized After Mid-Session Installation

**What broke:**

After the Delinea installer's Fix Issues button automatically installed IIS, the installer still showed IIS prerequisites as failed even though IIS was confirmed installed and running.

**Root cause:**

The Delinea installer loads the Microsoft.Web.Administration assembly at startup. When IIS is installed mid-session the assembly is already loaded in memory from the pre-IIS state. The installer cannot detect the new installation without reloading the assembly, which only happens when the installer process is fully restarted.

**How it was fixed:**

The installer was fully closed and reopened. On relaunch it loaded the assembly fresh, detected IIS correctly, and all prerequisites passed.

**What production looks like:**

When an installer installs a dependency mid-session always close and reopen the installer completely before continuing. In-memory state does not update when new system components are added during the same session. This applies to any enterprise installer that handles its own prerequisite installation.

---

## Issue 03: HTTP 503 Service Unavailable After VM Idle

**What broke:**

After leaving DC01 idle overnight Secret Server was unreachable at the expected URL. The browser returned HTTP Error 503 Service Unavailable.

**Root cause:**

IIS application pools have a default idle timeout of 20 minutes. When no requests are made for 20 minutes IIS stops the worker process to conserve resources. The SecretServer application pool stopped itself during the idle period and did not restart automatically when a new request came in.

**How it was fixed:**

IIS was restarted using iisreset. The idle timeout was then set to zero to prevent future timeouts:

```powershell
Set-ItemProperty "IIS:\AppPools\SecretServer" -Name processModel.idleTimeout -Value 00:00:00
```

**What production looks like:**

In production the IIS idle timeout is managed through server build standards and Group Policy. Application pools hosting critical services like a PAM vault are configured with idle timeout disabled and recycling schedules set to off-peak windows. A 503 in production on a PAM vault is a Severity 1 incident — engineers cannot access privileged credentials and all privileged work stops.

---

## Issue 04: Error Code 516 — Three Separate Root Causes

**What broke:**

The RDP launcher returned Error Code 516 Unable to connect three separate times after the protocol handler was installed. Each attempt failed for a different reason.

**Root cause and resolution — First occurrence:**

RDP was not enabled on DC01. Windows Server 2022 ships with RDP disabled by default. Enabled via PowerShell:

```powershell
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name fDenyTSConnections -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
```

**Root cause and resolution — Second occurrence:**

The secret in Secret Server was using a vault-generated password that did not match the actual Administrator password on DC01. The vault and the machine were out of sync. The secret was manually updated to match the actual machine password.

In production this is resolved by enabling Secret Server heartbeat and remote password changing so the vault manages rotation automatically and the two are never out of sync. When onboarding existing accounts the current password must be entered manually first before the vault can take over.

**Root cause and resolution — Third occurrence:**

Secret Server was installed on DC01 and the launcher was attempting to RDP from DC01 into itself. Windows blocks loopback RDP connections at the OS level by design. A machine cannot RDP into itself regardless of credentials or configuration.

A second VM was built — WIN-TARGET01 at 192.168.50.20, domain joined, RDP enabled — as a dedicated target. A new secret was created targeting WIN-TARGET01 and the launcher connected successfully.

**What production looks like:**

The PAM vault server is always on dedicated infrastructure separate from every managed system. This is not just a lab constraint — it is a core PAM architectural principle. The vault manages credentials for other machines. It is never one of those machines. Error Code 516 is a generic connectivity error that covers multiple failure modes — always work through them systematically rather than assuming the most complex cause first.

---

## Issue 05: License Gates Blocking JIT Workflow Features

**What broke:**

Three features required to complete the full JIT access workflow were blocked by the Delinea Secret Server free edition license: session recording configuration, Secret Policy creation, and the checkout interval setting.

**Root cause:**

The free edition is limited to one enabled user account and gates several enterprise workflow features behind a paid license. Secret Policy creation blocks the save operation with a license error. Session recording returns access denied regardless of role assignment. The checkout interval setting does not appear on the secret settings tab at all.

**How it was fixed:**

Each limitation was documented transparently with screenshots showing the correctly built configuration alongside the license gate error. The JIT access concept was demonstrated using the comment-required checkout gate which is available in the free edition — every access attempt required a typed business justification logged to the audit trail with a timestamp and identity attached.

**What production looks like:**

In production Secret Server Professional or Premium edition is used. Secret Policies enforce checkout intervals, automatic password rotation on check-in, and require approval workflows for sensitive secrets. Session recording captures every keystroke and mouse movement during a privileged session and stores it as a tamper-evident audit artifact. The architecture for all of these was built and documented to the point the license allowed. The concept is fully understood.

---

## What I Would Do Differently in Production

SQL Server and IIS would be pre-staged on the server before the Delinea installer runs. The installer would only need to connect to existing components rather than installing anything mid-session.

The IIS idle timeout would be set to zero and application pool settings would be documented in the server build standard as part of the initial configuration, not discovered post-deployment.

The PAM vault server would always be on dedicated infrastructure isolated from managed systems from the start. The architectural principle that the vault cannot manage itself is non-negotiable.

Remote password changing and heartbeat would be enabled for every secret from day one so vault credentials are always in sync with the machines they protect.

---

*Built by Justin Gallimore | [github.com/JustinGallimore](https://github.com/JustinGallimore)*
