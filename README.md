# 🛡️ SOC Analyst Home Lab

## Overview
A hands-on home lab simulating a real-world reverse shell attack and 
detection workflow using VirtualBox. Built to develop practical SOC 
analyst skills in attack simulation, traffic analysis, and log based 
detection.

## Lab Environment
| VM | Role | IP |
|---|---|---|
| Kali Linux | Attacker | 192.168.20.11 |
| Windows 10 | Victim Workstation | 192.168.20.10 |
| Windows Server 2022 | Log Server | 192.168.10.5 |
| Metasploitable 2 | Vulnerable Target | 192.168.10.20 |

## Tools Used
- VirtualBox —> virtualization
- Metasploit Framework —> exploitation
- msfvenom —> payload generation
- Nmap —> reconnaissance
- Wireshark —> traffic analysis
- Windows Event Viewer —> log analysis

## Phases
- [Phase 1 - Network Setup](./Phase-1-Network-Setup/notes.md)
- [Phase 2 - Reconnaissance](./Phase-2-Reconnaissance/notes.md)
- [Phase 3 - Payload Delivery](./Phase-3-Payload-Delivery/notes.md)
- [Phase 4 - Exploitation](./Phase-4-Exploitation/notes.md)
- [Phase 5 - Wireshark Analysis](./Phase-5-Wireshark-Analysis/notes.md)
- [Phase 6 - Log Analysis](./Phase-6-Log-Analysis/notes.md)

## ⚠️ Disclaimer
All activities conducted in a fully isolated virtual environment 
for educational purposes only.
