# windows-11-25h2-vbs-vmware-cpl0
windows 11 25H2 VBS/Hypervisor Conflict: VMware ULM → CPL0

Documented Windows 11 25H2 Home VBS/Hyper-V issue affecting VMware Workstation; includes evidence, diagnostics, rollback, and a machine-verified WindowsHello DeviceGuard workaround.

# Windows 11 25H2 VBS/Hypervisor Conflict: VMware ULM → CPL0

A documented troubleshooting case for **Windows 11 Home 25H2** where Virtualization-Based Security (VBS) and the Windows hypervisor remained active despite the usual Hyper-V/VBS disable steps, forcing VMware Workstation into a slower Hyper-V-backed monitor mode.

The case was resolved by disabling the **Device Guard `WindowsHello` scenario**, after which VBS stopped launching and VMware switched from **`Monitor Mode: ULM` to `Monitor Mode: CPL0`**.

> **Important:** This is an empirically verified workaround on one exact system. It is **not presented as a universal or Microsoft-supported fix**. It changes Windows security configuration and may affect Windows Hello/PIN behavior.

## Repository at a glance

**Status:** Resolved on the tested system  
**Date tested:** 2026-09-13  
**Primary goal:** VMware Workstation performance  
**Secondary goal:** Linux desktop / Wayland experimentation

### Tested system

| Component | Tested value |
|---|---|
| Laptop | Acer Nitro V 15 ANV15-41 |
| CPU | AMD Ryzen 5 7535HS |
| Host GPU | NVIDIA GeForce RTX 2050 4 GB |
| Host RAM | 24 GB |
| Windows | Windows 11 Home 25H2 |
| OS build | 26200.9445 |
| BIOS mode | UEFI |
| Secure Boot | On |
| VMware | Workstation Pro 26.0.1.25688693 |
| Guest | Kali Linux |
| Guest RAM | 8 GB |
| Guest vCPU | 4 |
| Guest graphics | VMware 3D acceleration enabled, 8 GB graphics memory |
| Guest graphics driver | `vmwgfx` 2.21.0.0 |
| Guest Mesa | 26.1.6-1 |

## The problem

The host had AMD hardware virtualization enabled. Windows Task Manager reported:

```text
Virtualization: Enabled
```

However, Windows Security showed **Memory Integrity = On** with:

```text
This setting is managed by your administrator.
```

`msinfo32` showed VBS running and reported that a hypervisor had been detected.

VMware then reported:

```text
IOPL_Init: Hyper-V detected by CPUID
Monitor Mode: ULM
```

This was undesirable for the use case because the goal was the best possible VMware responsiveness, especially for graphical Linux guests.

## Initial VBS state

Before the decisive change:

```text
VirtualizationBasedSecurityStatus = 2
CodeIntegrityPolicyEnforcementStatus = 2
SecurityServicesConfigured = {2}
SecurityServicesRunning = {2}
```

Interpretation:

- VBS was enabled and running.
- HVCI/Memory Integrity was configured and running.
- Code-integrity policy enforcement was active.

## What was tried first

The normal, lower-risk remediation steps were attempted before changing the Device Guard scenario:

1. Disable unused Windows virtualization features such as Hyper-V / Virtual Machine Platform / Windows Hypervisor Platform / Sandbox where applicable.
2. Disable the normal Memory Integrity/HVCI setting where the UI permits it.
3. Disable the Hyper-V boot launch path:

```powershell
bcdedit /set hypervisorlaunchtype off
```

4. Disable VSM launch:

```powershell
bcdedit /set vsmlaunchtype off
```

5. Disable the HVCI scenario:

```powershell
reg add "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\HypervisorEnforcedCodeIntegrity" /v Enabled /t REG_DWORD /d 0 /f
```

6. Disable the top-level VBS registry setting:

```powershell
reg add "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard" /v EnableVirtualizationBasedSecurity /t REG_DWORD /d 0 /f
```

These changes successfully stopped HVCI itself, but **VBS still launched** and Windows still reported that a hypervisor had been detected.

At that stage the host showed:

```text
VirtualizationBasedSecurityStatus = 2
CodeIntegrityPolicyEnforcementStatus = 2
SecurityServicesConfigured = {0}
SecurityServicesRunning = {0}
```

This was an important diagnostic point: HVCI was off, but the VBS environment itself was still being launched.

## Device Guard investigation

The following registry areas were inspected:

```powershell
reg query "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard"
reg query "HKLM\SOFTWARE\Policies\Microsoft\Windows\DeviceGuard"
reg query "HKLM\SYSTEM\CurrentControlSet\Control\CI\Config"
reg query "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios"
reg query "HKLM\SYSTEM\CurrentControlSet\Control\Lsa"
```

`CiTool.exe -lp -json` also showed Microsoft-signed system policies, including:

- `Microsoft Windows Virtualization Based Security Policy` — enforced
- `Microsoft Windows Endpoint Security Policy` — enforced
- `Microsoft Windows Driver Policy` — enforced

The important discovery was that the following Device Guard scenario was still enabled:

```powershell
reg query "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\WindowsHello"
```

Result:

```text
Enabled    REG_DWORD    0x1
```

## Decisive workaround

A backup was created first:

```powershell
reg export "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\WindowsHello" "$env:USERPROFILE\Desktop\WindowsHello-backup.reg" /y
```

Then the scenario was disabled:

```powershell
reg add "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\WindowsHello" /v Enabled /t REG_DWORD /d 0 /f
```

A full shutdown was performed:

```powershell
shutdown /s /t 0
```

After booting back into Windows, VBS was finally disabled.

## Verification: Windows side

The post-change check returned:

```text
VirtualizationBasedSecurityStatus = 0
CodeIntegrityPolicyEnforcementStatus = 2
SecurityServicesConfigured = {0}
SecurityServicesRunning = {0}
```

And:

```text
Virtualization-based security: Status: Not enabled
```

The previous `systeminfo` line:

```text
A hypervisor has been detected.
```

was no longer returned by the tested query.

### What this proves

On this specific Windows 11 25H2 Home installation, disabling the `WindowsHello` Device Guard scenario was the missing step that stopped VBS from launching.

The fact that this changed the machine from:

```text
VBS running
```

to:

```text
VBS not enabled
```

after reboot is the strongest evidence in this case.

## Verification: VMware side

A VMware log generated after the change reported:

```text
IOPL_VBSRunning: VBS is set to 0
Monitor Mode: CPL0
```

Before the change, VMware repeatedly reported:

```text
IOPL_Init: Hyper-V detected by CPUID
Monitor Mode: ULM
```

Therefore the workaround was not merely changing a Windows status display; it changed VMware's actual monitor mode.

### Before

```text
Windows VBS          ON
Windows hypervisor   RUNNING
VMware monitor mode  ULM
```

### After

```text
Windows VBS          OFF
Windows hypervisor   NOT LAUNCHED
VMware monitor mode  CPL0
```

For VMware performance, **CPL0 is the desired result**.

## Why this matters for VMware

When Hyper-V/VBS is active, VMware Workstation can operate through the Windows hypervisor path rather than its preferred direct virtualization path. VMware documents that this can introduce additional overhead and reduce VM performance.

In this case the VMware log provided direct evidence:

```text
ULM  →  CPL0
```

after disabling the Windows VBS path.

## BIOS: what was NOT changed

The Acer BIOS did not expose an obvious SVM/virtualization toggle.

That was **not** the problem.

Windows Task Manager already showed:

```text
Virtualization: Enabled
```

Therefore AMD hardware virtualization was already active.

**Do not disable AMD SVM/AMD-V in BIOS when troubleshooting this particular symptom.**

The target was the **Windows VBS/Hyper-V software layer**, not AMD hardware virtualization.

## Windows Hello warning

This workaround changes:

```text
HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\WindowsHello
```

and may affect Windows Hello/PIN behavior.

Before applying it, make sure you know your normal Windows account password.

Keep the registry backup until the machine has passed multiple normal sign-in/reboot cycles.

Rollback:

```powershell
reg import "$env:USERPROFILE\Desktop\WindowsHello-backup.reg"
```

Then reboot and verify VBS/hypervisor state again.

A broader Device Guard backup used during this investigation can also be restored with:

```powershell
reg import "$env:USERPROFILE\Desktop\DeviceGuard-backup.reg"
```

## Security trade-off

VBS, HVCI/Memory Integrity, and related Device Guard protections exist to harden Windows against certain kernel- and code-integrity attacks.

Disabling them is **not security-neutral**.

This case intentionally prioritized VMware performance on a personal development/virtualization host. Anyone reproducing it should understand the security trade-off before applying the workaround.

## Why I am calling this a workaround, not a universal fix

This repository documents **one reproducible machine-specific case**.

It does **not** establish that:

```text
WindowsHello Enabled = 0
```

is always the correct solution for every Windows 11 25H2 machine.

Windows 11 security policy behavior can vary by edition, build, device configuration, firmware, Microsoft policy state, and enabled security features.

The correct approach for future users is:

1. Confirm hardware virtualization is already enabled.
2. Record the actual VBS/HVCI/hypervisor state.
3. Try the normal Microsoft/VMware disable procedure first.
4. Inspect Device Guard scenarios.
5. Back up before registry changes.
6. Use the `WindowsHello` scenario as a diagnostic lead only when the evidence matches this case.
7. Verify the result after reboot.
8. Verify VMware monitor mode directly from `vmware.log`.

## Reproduction checklist

Run:

```powershell
msinfo32
```

Then:

```powershell
Get-CimInstance Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard |
  Format-List *
```

Then:

```powershell
CiTool.exe -lp -json
```

Inspect:

```powershell
reg query "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\WindowsHello"
```

If the result is:

```text
Enabled = 0x1
```

and the machine otherwise matches the case described here, back up the key before testing the workaround.

After reboot, verify:

```powershell
Get-CimInstance Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard |
  Select-Object VirtualizationBasedSecurityStatus,
                CodeIntegrityPolicyEnforcementStatus,
                SecurityServicesConfigured,
                SecurityServicesRunning
```

and:

```powershell
systeminfo | Select-String "Virtualization-based security","hypervisor has been detected"
```

For VMware, start a VM and inspect its log:

```powershell
Select-String -Path "C:\path\to\vmware.log" -Pattern "Monitor Mode:"
```

Desired result:

```text
Monitor Mode: CPL0
```

## Linux / Omarchy note

The Windows virtualization fix does **not** automatically solve VMware + Wayland/Hyprland compatibility.

The tested Kali guest correctly detected:

```text
VMware SVGA II Adapter
Kernel driver: vmwgfx
Mesa: 26.1.6-1
OpenGL renderer: SVGA3D
```

However, current 2026 Omarchy / Hyprland / Aquamarine reports document separate VMware `vmwgfx` problems, including black-screen-with-cursor behavior and Wayland client failures with accelerated graphics.

Treat these as a **separate guest graphics compatibility layer**.

## Privacy when publishing logs

Raw VMware logs can contain:

- Windows usernames
- VM paths
- VM names
- local IP addresses
- hostnames

Redact these before uploading logs or screenshots to GitHub.

## Evidence

The incident was verified using:

- Windows `msinfo32`
- `Win32_DeviceGuard`
- `CiTool.exe -lp -json`
- Windows registry inspection
- VMware Workstation 26.0.1 logs
- Kali `lspci`, `lsmod`, `inxi`, `glxinfo`, and `eglinfo`

The strongest evidence is the before/after transition:

```text
Before:
VBS = 2
Hypervisor detected
VMware = ULM

After:
VBS = 0
Hypervisor not detected
VMware = CPL0
```

## References

### Microsoft

- [Enable virtualization-based protection of code integrity](https://learn.microsoft.com/en-us/windows/security/hardware-security/enable-virtualization-based-protection-of-code-integrity)
- [VirtualizationBasedTechnology Policy CSP](https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-virtualizationbasedtechnology)
- [Microsoft Q&A: Windows 11 25H2 — Hypervisor detected and VBS running](https://learn.microsoft.com/en-us/answers/questions/5925133/windows-11-25h2-hypervisor-detected-and-vbs-runnin)
- [Microsoft Q&A: VBS not disabled on Windows 11 24H2](https://learn.microsoft.com/en-us/answers/questions/2113711/vbs-is-not-disabled-on-windows-11-version-24h2-os)

### VMware / Broadcom

- [KB 389469 — VMware monitor mode / CPL0 vs Hyper-V-backed mode](https://knowledge.broadcom.com/external/article/389469)
- [KB 417896 — Windows 11 VBS/Hyper-V Workstation performance](https://knowledge.broadcom.com/external/article/417896/after-the-host-os-upgrade-to-version-win.html)
- [KB 315385 — VMware Workstation and Hyper-V compatibility](https://knowledge.broadcom.com/external/article/315385)

### Linux / Omarchy / Hyprland

- [Kali Linux — Install Kali in VMware](https://www.kali.org/docs/virtualization/install-vmware-guest-vm/)
- [Omarchy #8113 — VMware Workstation + 3D acceleration](https://github.com/omacom/omarchy/issues/8113)
- [Aquamarine #360 — vmwgfx dmabuf/import issue](https://github.com/hyprwm/aquamarine/issues/360)
- [Omarchy discussion #572 — Omarchy on VMware Workstation](https://github.com/omacom/omarchy/discussions/572)

## Suggested issue title for follow-up reports

> Windows 11 25H2 Home: VBS remains running after standard Hyper-V disable steps; WindowsHello DeviceGuard scenario was the remaining trigger

## Suggested repository topics

```text
windows-11
windows-11-25h2
vbs
deviceguard
hyper-v
vmware
vmware-workstation
virtualization
hvci
windowshello
kali-linux
wayland
hyprland
omarchy
```

## Suggested short repository description

> Documented Windows 11 25H2 Home VBS/Hyper-V issue affecting VMware Workstation; includes evidence, diagnostics, rollback, and a machine-verified WindowsHello DeviceGuard workaround.

## License suggestion

For a documentation/research repository, **CC BY 4.0** is a good fit if you want others to freely reuse the write-up with attribution. A code-oriented license such as MIT is better only if the repository later contains scripts/tools intended for reuse.
