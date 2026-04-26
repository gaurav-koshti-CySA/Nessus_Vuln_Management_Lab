# Network Configuration Commands
## Vulnerability Management Lab Setup
**Author:** Gaurav Koshti  
**Date:** April 25, 2026  
**Lab Environment:** VirtualBox NAT Network (`LabNetwork` — `10.0.2.0/24`)

---

## Lab VM Details

| VM | Role | IP Address |
|---|---|---|
| Windows Server | Nessus Scanner | 10.0.2.15 |
| Windows 11 Endpoint | Scan Target | 10.0.2.122 |

---

## Commands Run on Windows Server (Scanner)

### 1. Enable Remote Registry Service
Allows Nessus to remotely access the Windows registry for credentialed scanning.
```cmd
sc config RemoteRegistry start= auto
net start RemoteRegistry
```
**Verify it's running:**
```cmd
sc query RemoteRegistry
```

### 2. Enable File and Printer Sharing
Required for Nessus to authenticate and communicate over SMB with target machines.
```cmd
netsh advfirewall firewall set rule group="File and Printer Sharing" new enable=Yes
```

### 3. Enable Network Discovery
Allows the scanner to discover and communicate with other devices on the network.
```cmd
netsh advfirewall firewall set rule group="Network Discovery" new enable=Yes
```

### 4. Enable SMB (Server Message Block)
Ensures the LanmanServer service is running for file sharing and remote access.
```cmd
sc config LanmanServer start= auto
net start LanmanServer
```

### 5. Enable Windows Management Instrumentation (WMI)
WMI is critical for credentialed scanning — Nessus uses it to query system info,
installed software, patches, and configurations remotely.
```cmd
netsh advfirewall firewall set rule group="Windows Management Instrumentation (WMI)" new enable=Yes
```

---

## Commands Run on Windows 11 Endpoint (Scan Target)

### 1. Allow ICMP (Ping) — Enable connectivity testing
```cmd
netsh advfirewall firewall add rule name="Allow ICMPv4" protocol=icmpv4:8,any dir=in action=allow profile=any
```
**Alternative — Disable firewall for lab testing:**
```cmd
netsh advfirewall set allprofiles state off
```
> Note: Firewall was disabled on the endpoint for lab purposes only.
> In a production environment, specific rules would be configured instead.

### 2. Enable Remote Registry Service
```cmd
sc config RemoteRegistry start= auto
net start RemoteRegistry
```

### 3. Enable File and Printer Sharing
```cmd
netsh advfirewall firewall set rule group="File and Printer Sharing" new enable=Yes
```

### 4. Enable Network Discovery
```cmd
netsh advfirewall firewall set rule group="Network Discovery" new enable=Yes
```

### 5. Enable SMB
```cmd
sc config LanmanServer start= auto
net start LanmanServer
```

### 6. Enable WMI
```cmd
netsh advfirewall firewall set rule group="Windows Management Instrumentation (WMI)" new enable=Yes
```

---

## Additional Credentialed Scan Preparation (Windows 11 Endpoint)

### Enable Local Account Token Filter Policy
Without this, Nessus gets "Access Denied" even with correct credentials on Windows 10/11.
```cmd
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" /v LocalAccountTokenFilterPolicy /t REG_DWORD /d 1 /f
```

### Verify Admin Shares are Active
```cmd
net share
```
**Expected output:**
```
C$        C:\          Default share
IPC$                   Remote IPC
ADMIN$    C:\WINDOWS   Remote Admin
```

### WinVerifyTrust Registry Fix (CVE-2013-3900)
```cmd
reg add "HKLM\Software\Microsoft\Cryptography\Wintrust\Config" /v EnableCertPaddingCheck /t REG_DWORD /d 1 /f
reg add "HKLM\Software\Wow6432Node\Microsoft\Cryptography\Wintrust\Config" /v EnableCertPaddingCheck /t REG_DWORD /d 1 /f
```

---

## Why These Commands Matter

| Service/Rule | Purpose in Vulnerability Scanning |
|---|---|
| **Remote Registry** | Nessus reads registry keys to detect software versions, patches, and misconfigurations |
| **File & Printer Sharing** | Enables SMB-based authentication for credentialed scans |
| **Network Discovery** | Allows scanner to locate and communicate with target hosts |
| **LanmanServer (SMB)** | Core Windows file sharing service — required for remote access |
| **WMI** | Nessus queries WMI to enumerate installed software, running services, OS info, and patch levels |
| **ICMP (Ping)** | Basic connectivity testing between scanner and target |

---

*This configuration was performed as part of a home lab vulnerability management project
using Tenable Nessus Essentials. All changes were made in an isolated VirtualBox environment.*
