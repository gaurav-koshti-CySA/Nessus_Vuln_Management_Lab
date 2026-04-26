# Vulnerability Management Lab — Tenable Nessus Essentials

**Platform:** VirtualBox | **Scanner:** Tenable Nessus Essentials 10.12.0 | **Date:** April 2026

---

## Overview

This project documents a full vulnerability management lifecycle built in a home lab environment using VirtualBox and Tenable Nessus Essentials. The goal was to simulate what vulnerability management looks like in a real environment — from deploying the scanner and configuring credentialed access, to introducing known-vulnerable software, running progressive scans, remediating findings, and validating the fixes through re-scanning.

Six scans were conducted in total, including a credentialed vs. uncredentialed comparison to demonstrate why credential-based scanning is essential in any enterprise VM program.

---

## Lab Environment

| VM | OS | IP | Role |
|---|---|---|---|
| Windows Server 2019 | Windows Server Standard | 10.0.2.15 | Nessus Scanner |
| Windows 11 Endpoint | Windows 11 Pro | 10.0.2.122 | Scan Target |

Both VMs were connected via a VirtualBox NAT Network (`LabNetwork — 10.0.2.0/24`) with DHCP enabled to allow VM-to-VM communication.

---

## Scanner Setup

Nessus Essentials was installed on the Windows Server VM. After installation, the scanner took approximately 45 minutes to download and compile over 85,000 plugins on first run. The service was configured to start automatically and was accessible via `https://localhost:8834`.

![Nessus Dashboard](screenshots/11_nessus_plugins_done_compiling.png)

---

## Target Configuration

Before running any credentialed scans, the Windows 11 endpoint required several configuration changes to allow Nessus remote access:

- Remote Registry service enabled and set to automatic start
- File and Printer Sharing enabled via Windows Firewall
- Network Discovery enabled
- WMI (Windows Management Instrumentation) firewall rules enabled
- SMB (LanmanServer) service confirmed running
- Admin shares (C$, ADMIN$, IPC$) verified active
- `LocalAccountTokenFilterPolicy` registry key set to 1 to allow remote admin access with local accounts

All configuration commands are documented in [`scripts/network_config_commands.md`](scripts/network_config_commands.md).

![Admin Shares Verified](screenshots/09_win11_admin_shares_verified.png)

---

## Vulnerable Software Used

| Application | Vulnerable Version | Patched Version | CVEs |
|---|---|---|---|
| Notepad++ | 7.5.4 | 8.9.3 | Buffer Overflow, Arbitrary Code Execution, DLL Hijacking, Privilege Escalation |
| VLC Media Player | 2.2.0 | 3.0.23 | Multiple CRITICAL CVEs — CVSS up to 9.8 |

Both installers were staged in `C:\VulnLab\` on the endpoint before scanning began.

![VulnLab Folder](screenshots/08_vulnlab_all_installers.png)

---

## Scan Summary

| Scan | Description | Notepad++ | VLC | Teams |
|---|---|---|---|---|
| Baseline | Clean endpoint, no extra software | - | - | Installed |
| Scan 2 | Notepad++ 7.5.4 installed | 7.5.4 | - | Installed |
| Scan 3 | VLC 2.2.0 added | 7.5.4 | 2.2.0 | Installed |
| Scan 4 | VLC patched, Teams removed | 7.5.4 | 3.0.23 | Removed |
| Scan 5 | Full remediation | 8.9.3 | 3.0.23 | Removed |
| Uncredentialed | Comparison scan — no credentials | 8.9.3 | 3.0.23 | Removed |

![All Scans Overview](screenshots/32_all_scans_overview.png)

---

## Scan Results

### Scan 1 — Baseline

The baseline scan ran against a clean Windows 11 endpoint with no additional software installed. Nessus identified 42 findings — 2 High, 1 Low, and the rest informational.

Key findings:
- **HIGH (CVSS 8.8)** — WinVerifyTrust Signature Validation CVE-2013-3900
- **HIGH (CVSS 8.1)** — Microsoft Outlook missing security update KB5002574

![Baseline Results](screenshots/14_scan1_baseline_results.png)

---

### Scan 2 — Notepad++ 7.5.4 Installed

After installing Notepad++ 7.5.4 on the endpoint, the scan identified 6 additional High severity vulnerabilities directly tied to the outdated application.

Key findings:
- Notepad++ Buffer Overflow (CVSS 7.8)
- Notepad++ Arbitrary Code Execution (CVSS 7.8)
- Notepad++ DLL Hijacking (CVSS 7.8)
- Notepad++ Privilege Escalation CVE-2025-49144 (CVSS 7.3)

Nessus recommended upgrading to version 8.9.2 or later.

![Scan 2 Results](screenshots/19_scan2_npp754_results.png)

---

### Scan 3 — Notepad++ 7.5.4 + VLC 2.2.0 Installed

Adding VLC 2.2.0 significantly increased the risk profile of the endpoint. The scan returned 5 Critical and 4 High severity findings tied to VLC alone — all with CVSS scores of 9.1 or higher.

Key VLC findings:
- VLC Multiple Vulnerabilities (CVSS 9.8) — affects versions before 3.0.7, 3.0.8, 3.0.9
- VLC Overflow Condition (CVSS 9.8)
- VLC Denial of Service (CVSS 9.1)
- VLC Use-After-Free RCE (CVSS 8.0, VPR 9.0)

Nessus recommended upgrading VLC to 3.0.22 or later to address 40 vulnerabilities in a single remediation action.

![Scan 3 VLC Results](screenshots/23_scan3_vlc_vulnerabilities.png)
![Scan 3 NPP Results](screenshots/24_scan3_npp_vulnerabilities.png)

---

### Scan 4 — Partial Remediation (VLC Patched, Teams Removed)

VLC was upgraded to 3.0.23 and Microsoft Teams desktop was uninstalled. The rescan confirmed that all VLC-related Critical findings were resolved, dropping Criticals from 5 to 0.

Notepad++ 7.5.4 remained installed, so those High severity findings persisted.

**Note:** Even after uninstalling Microsoft Teams, it continued appearing in the Remediations tab. Teams left behind residual registry keys and WindowsApps metadata that Nessus continued to detect. In an enterprise environment this would typically require Microsoft's official cleanup tool or manual registry remediation. This was documented as a known real-world limitation rather than a failure of the remediation process.

![Scan 4 Results](screenshots/27_scan4_partial_results.png)

---

### Scan 5 — Full Remediation

Notepad++ was upgraded from 7.5.4 to 8.9.3. The WinVerifyTrust registry fix was applied manually:

```cmd
reg add "HKLM\Software\Microsoft\Cryptography\Wintrust\Config" /v EnableCertPaddingCheck /t REG_DWORD /d 1 /f
reg add "HKLM\Software\Wow6432Node\Microsoft\Cryptography\Wintrust\Config" /v EnableCertPaddingCheck /t REG_DWORD /d 1 /f
```

Microsoft Outlook was also uninstalled to address the remaining Outlook security update finding.

The final scan showed 0 Critical and 1 High finding remaining — the residual Teams artifact mentioned above.

![Scan 5 Final Results](screenshots/30_scan5_final_results.png)

---

## Risk Progression

| Scan | Total | Critical | High | Low |
|---|---|---|---|---|
| Baseline | 42 | 0 | 2 | 1 |
| Scan 2 — NPP 7.5.4 | 44 | 0 | 8 | 1 |
| Scan 3 — NPP + VLC | 46 | 5 | 12 | 1 |
| Scan 4 — Partial Fix | 45 | 0 | 8 | 1 |
| Scan 5 — Full Fix | 45 | 0 | 1 | 1 |

From peak risk (Scan 3) to final state (Scan 5): Critical findings reduced by 100%, High findings reduced by 92%.

---

## Credentialed vs. Uncredentialed Scanning

| Metric | Uncredentialed | Credentialed (Baseline) |
|---|---|---|
| Total Findings | 17 | 42 |
| Critical | 0 | 0 |
| High | 0 | 2 |
| Software Detected | No | Yes |
| Patch Level Assessed | No | Yes |
| Scan Duration | 8 minutes | 32 minutes |

Without credentials, Nessus missed 60% of findings and had no visibility into installed software versions or patch status. This is why credentialed scanning is the standard in enterprise vulnerability management programs.

![Uncredentialed Scan](screenshots/33_uncredentialed_scan_results.png)

---

## Real-World Observation — Teams Remediation

Microsoft Teams was uninstalled via Windows Settings. However, it continued appearing in the Remediations tab across subsequent scans. Nessus was detecting residual registry keys and WindowsApps metadata left behind after the uninstall. In an enterprise environment, this would require Microsoft's official Teams cleanup tool or manual registry cleanup to fully resolve. This was noted as a known limitation rather than a scan failure.

---

## Remediation SLA Reference

| Severity | Industry SLA | Applied In This Lab |
|---|---|---|
| Critical | 24-48 hours | Scan 4 (same session) |
| High | 7 days | Scan 5 (same session) |
| Medium | 30 days | N/A |
| Low | 90 days | Documented, not remediated |

---

## MITRE ATT&CK Mapping

| Vulnerability | Technique | Tactic |
|---|---|---|
| DLL Hijacking (Notepad++) | T1574.001 — DLL Search Order Hijacking | Persistence, Privilege Escalation |
| Privilege Escalation (Notepad++) | T1068 — Exploitation for Privilege Escalation | Privilege Escalation |
| VLC Use-After-Free RCE | T1203 — Exploitation for Client Execution | Execution |
| WinVerifyTrust Bypass | T1553.002 — Code Signing | Defense Evasion |
| Outlook NTLM Hash Leak | T1557 — Adversary-in-the-Middle | Credential Access |

---

## Lessons Learned

Credentialed scanning is not optional — the gap between 17 and 42 findings makes the case better than any documentation. Without credentials, you simply don't know what's on your endpoints.

Remediation is rarely clean. Uninstalling Teams through the UI did not fully remove its footprint. Registry artifacts persisted and continued appearing in scan results. Real remediation often requires more than clicking uninstall.

Plugin compilation takes time and resources. On a VM with 3 CPU cores, the initial compilation took close to 45 minutes. In production, scanner sizing matters.

Re-scanning is the only way to confirm remediation. Assuming a patch worked without a follow-up scan is how things get missed.

---

## Repository Structure

```
Nessus_Vuln_Management_Lab/
├── README.md
├── screenshots/          # All 33 scan screenshots in order
└── scripts/
    └── network_config_commands.md
```

---

## Tools Used

- Tenable Nessus Essentials 10.12.0
- Oracle VirtualBox 7.x
- Windows Server 2019 (Scanner)
- Windows 11 Pro (Target)
- Notepad++ 7.5.4 and 8.9.3
- VLC Media Player 2.2.0 and 3.0.23

---

*Gaurav Koshti | Montreal, Canada | [LinkedIn](https://linkedin.com/in/gaurav-koshti)*
