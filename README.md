# 🛡️ SOC Analyst Home Lab

## 📋 Overview
A hands-on home lab simulating a real world reverse shell attack and 
detection workflow using VirtualBox. Built to develop practical SOC 
analyst skills in attack simulation, network traffic analysis and 
host based log forensics. Every phase of this lab was documented as 
it happened including the troubleshooting steps and failures.

## 🖥️ Lab Environment
| VM | Role | IP Address |
|---|---|---|
| Kali Linux | Attacker | 192.168.1.17 |
| Windows 10 | Victim Workstation | 192.168.1.18 |
| Windows Server 2022 | Log Collector | 192.168.1.222 |
| Metasploitable 2 | Vulnerable Target | 192.168.1.21 |

## 🛠️ Tools Used
- VirtualBox for virtualization and network isolation
- Metasploit Framework for exploitation and post exploitation
- msfvenom for malicious payload generation
- Nmap for network reconnaissance and port scanning
- Wireshark for live network traffic capture and analysis
- Windows Event Viewer for host based log forensics
- Windows Event Forwarding for centralized log collection attempt

## 👾 Attack Summary
A reverse shell payload named Free_VPN.exe was created using msfvenom 
and delivered to the Windows 10 victim machine simulating a real world 
phishing attack. The filename was chosen deliberately as a social 
engineering lure since free VPN software is something many people 
download without thinking twice. The payload established a meterpreter 
session back to Kali Linux giving full remote access to the victim 
machine. The entire attack was then analyzed from the defender 
perspective using Wireshark for network forensics and Windows Event 
Viewer for host based forensics.

## 📁 Phases
- [Phase 1 - Network Setup](./Phase-1-Network-Setup/notes.md)
- [Phase 2 - Reconnaissance](./Phase-2-Reconnaissance/notes.md)
- [Phase 3 - Payload Creation and Delivery](./Phase-3-Payload-Delivery/notes.md)
- [Phase 4 - Wireshark Traffic Analysis](./Phase-4-Wireshark-Analysis/notes.md)
- [Phase 5 - Windows Event Log Analysis](./Phase-5-Log-Analysis/notes.md)
- [Phase 6 - Log Forwarding](./Phase-6-Log-Forwarding/notes.md)

## 🔍 Key Findings
Nmap reconnaissance revealed open ports 135, 139 and 445 on the victim 
machine with SMB running and message signing not required which in a 
real environment would be flagged as a relay attack risk.

The reverse shell payload was initially caught and killed by Windows 
Defender the moment it executed which is realistic behavior showing 
how antivirus detects known malware signatures. Windows Defender was 
disabled to complete the simulation.

Wireshark captured malware beaconing behavior on port 4444 showing 
Free_VPN.exe repeatedly attempting to connect back to Kali even before 
the listener was active. Once the session was established active 
meterpreter traffic was visible with fully encrypted payloads flowing 
in both directions.

Windows Event ID 4688 successfully logged the exact execution of 
Free_VPN.exe including the full file path, the parent process 
explorer.exe confirming the user double clicked it and the username 
wsp vro. Post exploitation commands run through the meterpreter shell 
were also captured in the logs proving the complete attack chain from 
initial execution to active attacker control.

## 🧭 MITRE ATTCK Mapping

| Technique ID | Technique Name | How It Was Used |
|---|---|---|
| T1566 | Phishing | Free_VPN.exe delivered simulating a social engineering lure |
| T1059 | Command and Scripting Interpreter | Shell command used to access cmd.exe on victim machine |
| T1571 | Non Standard Port | Meterpreter C2 communication over port 4444 |
| T1082 | System Information Discovery | sysinfo gathered OS version and architecture |
| T1033 | System Owner Discovery | whoami confirmed the compromised user account |
| T1070 | Indicator Removal | Windows Defender disabled before payload execution |

## 📚 Lessons Learned
Windows 10 does not enable process creation auditing by default which 
means the first time I looked for evidence in the event logs it was 
not there. Audit policies must be configured proactively before an 
incident happens otherwise host based evidence will simply not exist 
when you need it most.

Malware beaconing behavior is visible in network traffic even when no 
active session exists. This means network monitoring alone can detect 
a compromise before an analyst even checks the host logs.

Windows Event Forwarding requires proper Active Directory and DNS 
infrastructure to function correctly. Attempting it in a mixed 
workgroup and domain environment will fail at the authentication layer 
regardless of network connectivity. This is a lesson that applies 
directly to real enterprise environments where infrastructure gaps 
can silently break security tooling.

## ⚠️ Disclaimer
All activities were conducted in a fully isolated VirtualBox internal 
network environment for educational purposes only. No real systems 
were affected and no traffic left the lab environment at any point.

## 📜 License

© 2026 Moaiad Eljack. All rights reserved.

This project and all its documentation were created and written by 
Moaiad Eljack as an original hands-on cybersecurity home lab. 
You are welcome to use this project as inspiration for your own lab 
but please do not copy the documentation or present it as your own work.
