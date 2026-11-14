# Wazuh Hands-On Project: Endpoint Monitoring, Malware Detection & Vulnerability Scanning

> A hands-on build of Wazuh's core detection capabilities in a home lab: file integrity monitoring, malware detection through VirusTotal, and vulnerability detection on a Windows 11 endpoint.

**Author:** Morakinyo

---

## Overview

This project deploys Wazuh in a home lab and uses it to demonstrate three core security-monitoring capabilities against a live Windows endpoint. Each part is backed by screenshot evidence and written up as a question-and-answer set (Q1–Q15).

| Part | Capability | What it demonstrates |
|------|-----------|----------------------|
| 1 | **File Integrity Monitoring (FIM)** | Real-time detection of changes to critical system files using syscheck |
| 2 | **Malware Detection via VirusTotal** | New files scanned against 70+ AV engines through Wazuh's integration |
| 3 | **Vulnerability Detection** | The endpoint's installed software scanned against known CVEs |

## Lab Environment

| Component | Detail |
|-----------|--------|
| Platform | Wazuh v4.14.7 |
| Manager | Wazuh OVA running as a VirtualBox VM (bridged network adapter) |
| Dashboard | `https://172.20.10.3` |
| Agent | Windows 11 Pro, native Wazuh agent |
| Agent name / IP | `Morakinyolastborn` / `172.20.10.2` |
| Agent deployment | Dashboard's **Deploy new agent** PowerShell script |

The agent deployment script needs the manager IP, agent name, group, and target OS.

---

## Part 1: File Integrity Monitoring

FIM tracks files and directories for unauthorized changes (creation, modification, deletion, permission changes) by comparing live file hashes against a trusted baseline.

**Why it matters in enterprise security**

- **Detects tampering:** catches malware replacing legitimate binaries or rootkits modifying system files
- **Supports compliance:** PCI-DSS, HIPAA and similar frameworks require FIM controls
- **Builds an audit trail:** shows who or what changed a file and when, which is critical for incident response
- **Catches insider threats:** flags privilege misuse that signature-based AV tools might miss

### Monitoring `C:\Windows\System32`

System32 holds core Windows binaries and DLLs that run with SYSTEM-level privilege and are implicitly trusted. Malware targets it through DLL hijacking or by replacing legitimate files, so it blends in and persists across reboots.

**Test:** a test file was added to System32.

| Result | Value |
|--------|-------|
| Event type | `added` |
| File | `c:\windows\system32\finaltest3.txt` |
| Rule | 554 |
| Level | 5 |

Wazuh logged the FIM event within minutes of the file appearing, confirming near real-time detection inside a live system folder.

---

## Part 2: Malware Detection via VirusTotal

VirusTotal is a free Google-owned service that checks files, URLs and hashes against 70+ antivirus engines and threat-intel feeds at once. The consensus verdict is more reliable than any single AV product.

### Integration (`ossec.conf`, manager)

```xml
<integration>
  <name>virustotal</name>
  <api_key>YOUR_API_KEY</api_key>
  <group>syscheck</group>
  <alert_format>json</alert_format>
</integration>
```

`<group>syscheck</group>` scopes the integration to FIM alerts only. VirusTotal lookups therefore happen solely for newly added or modified files, not for every alert Wazuh generates.

Applied with:

```bash
sudo systemctl restart wazuh-manager
```

> ⚠️ Never commit a real API key. The key above is a placeholder.

### Use case: detecting the EICAR test file

The EICAR string is a harmless, industry-standard test signature recognized by every antivirus engine. It validates detection pipelines without using real malware.

1. Temporarily disabled Windows Defender real-time protection so the test file could reach disk
2. Downloaded the standard EICAR anti-malware test file
3. Saved it as `eicar.txt` in the user's Documents folder
4. Added Documents to the agent's monitored syscheck paths
5. Restarted the Wazuh agent to apply the new path

### Result: a two-stage detection

| Alert | Rule | Outcome |
|-------|------|---------|
| FIM | 550 / 554 | "Integrity checksum changed" / "File added" |
| VirusTotal | n/a | `malicious: 1`, `positives: 46`, verdict from 46 of 70+ engines |

The VirusTotal permalink and SHA-1 hash were also captured in the alert for traceability to the original scan report.

> ⚠️ Re-enable Windows Defender real-time protection as soon as testing is done, and only run this kind of test in an isolated lab.

---

## Part 3: Vulnerability Detection

### CVE and CVSS fundamentals

- **CVE:** a unique ID (e.g. `CVE-2024-43599`) for a publicly known vulnerability, maintained by MITRE, so every tool and team references the same issue
- **CVSS:** a 0.0–10.0 scoring framework rating how exploitable a vulnerability is and how severe its impact would be, used to prioritize patching
- **"High" severity:** a CVSS v3.x score of 7.0–8.9; 9.0 and above is Critical

### Enabling the vulnerability detector

```xml
<vulnerability-detector>
  <enabled>yes</enabled>
  <interval>5m</interval>
  <min_full_scan_interval>6h</min_full_scan_interval>
  <run_on_start>yes</run_on_start>
</vulnerability-detector>
```

| Setting | Purpose |
|---------|---------|
| `interval` (5m) | How often Wazuh checks for CVE feed updates and runs a lightweight scan |
| `min_full_scan_interval` (6h) | Minimum wait between full, resource-intensive scans of all agents |

### Findings: 501 vulnerabilities

| Critical | High | Medium | Low |
|:--------:|:----:|:------:|:---:|
| 6 | 338 | 151 | 6 |

**Sample high-severity findings from this endpoint**

| CVE | Description | CVSS |
|-----|-------------|:----:|
| CVE-2024-43599 | RDP Client remote code execution | 8.8 |
| CVE-2024-43584 | Scripting Engine security feature bypass | 8.4 |
| CVE-2024-43574 | Speech API remote code execution | 8.3 |
| CVE-2024-43582 | RDP Server remote code execution | 8.1 |

### What Wazuh reports per vulnerability

- CVE ID and vulnerability title
- Severity rating and numeric CVSS score
- CVSS metric breakdown (attack vector, complexity, impact)
- Affected package name and installed version
- Publish date and source (e.g. NVD)
- Direct reference link to the full CTI / NVD record

---

## Key Takeaways

1. **File Integrity Monitoring:** caught a file added directly inside `C:\Windows\System32` in near real time
2. **Malware Detection:** the VirusTotal integration confirmed the EICAR file as malicious, flagged by 46 engines, fully automated
3. **Vulnerability Detection:** surfaced 501 real CVEs on the endpoint, including multiple High-severity RCEs

## Skills Demonstrated

- SIEM deployment and agent management (Wazuh)
- File integrity monitoring configuration (syscheck)
- Threat-intelligence integration (VirusTotal API)
- Vulnerability management and CVSS-based prioritization
- Detection testing with safe, standard artifacts (EICAR)

## Repository Contents

```
.
├── README.md
└── Wazuh_Project_Evidence_Presentation.pptx   # Evidence deck with screenshots
```

Full question-and-answer detail (Q1–Q15) accompanies the deck as supporting documentation.

## Author

**Morakinyo**
---

*Built in an isolated lab environment for educational purposes.*
