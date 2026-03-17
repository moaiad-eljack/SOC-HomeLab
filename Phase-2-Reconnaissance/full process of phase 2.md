# Phase 2  Reconnaissance

## What I'm Trying to Do
Before launching any attack the first thing an attacker does is gather 
information about the target. This phase is about understanding what is 
running on the Windows 10 machine, what ports are open, what services 
are exposed and what operating system it is running. All of this is done 
passively from Kali without touching the target machine at all.

This is exactly what a real threat actor would actually do before deciding how to 
attack a system and it is also what a SOC analyst needs to understand 
in order to know what an attacker could see from the outside.

## Tool Used
Nmap (Network Mapper) version 7.98 running on Kali Linux. Nmap works 
by sending crafted network packets to a target and analyzing the 
responses to identify open ports, running services, software versions 
and the operating system.

## Target
Windows 10 VM at 192.168.1.18

## Scan 1  Aggressive Scan

### Command Used
```bash
nmap -A 192.168.1.18 -Pn
```

### What Each Flag Does
- nmap is the scanning tool
- A runs an aggressive scan which includes OS detection, service 
  version detection, script scanning and traceroute all in one command
- 192.168.1.18 is the target IP address
- Pn skips the ping check and assumes the host is up. Even though 
  Windows Firewall was disabled on the victim machine earlier in the 
  lab, using Pn is good practice in real engagements because you 
  cannot always guarantee a target will respond to ping. Keeping it 
  in the command ensures the scan runs regardless.

### Results
```
PORT     STATE  SERVICE        VERSION
135/tcp  open   msrpc          Microsoft Windows RPC
139/tcp  open   netbios-ssn    Microsoft Windows netbios-ssn
445/tcp  open   microsoft-ds   Microsoft SMB
```

### OS Detection
Nmap successfully fingerprinted the operating system as Microsoft 
Windows 10 version 1709 to 21H2. It also identified the machine 
hostname as DESKTOP-2KU324L and confirmed it is running inside a 
VirtualBox environment based on the MAC address prefix.

### SMB Finding
Port 445 was the most interesting result. SMB (Server Message Block) 
is the Windows file sharing protocol and has historically been the 
target of some of the most damaging attacks in history including 
WannaCry in 2017 which used the EternalBlue exploit against this 
exact port. Nmap also flagged that SMB signing was enabled but not 
required which in a real environment opens the door to relay attacks.

## Scan 2 Full Port Scan

### Command Used
```bash
nmap -sV -p- 192.168.1.18 -Pn
```

### What Each Flag Does
- sV enables service version detection to identify exact software 
  versions running on each port
- p- tells Nmap to scan all ports (around 6000 ports) instead of just the default 
  1000 most common ones
- This scan took around 2 minutes to complete compared to the first 
  scan which finished in under 20 seconds

### Additional Ports Found
```
PORT       STATE  SERVICE
5040/tcp   open   unknown
7680/tcp   open   pando-pub (Windows Update Delivery Optimization)
49664/tcp  open   msrpc
49665/tcp  open   msrpc
49666/tcp  open   msrpc
49667/tcp  open   msrpc
49668/tcp  open   msrpc
49669/tcp  open   msrpc
49670/tcp  open   msrpc
```

### What These Mean
Port 5040 Shows as unknown This is typically normal.
Port 7680 is used by Windows Update Delivery Optimization which allows 
Windows machines to share update files with each other on the same 
network. The high numbered ports from 49664 to 49670 are Windows 
dynamic RPC ports that get assigned automatically when Windows services 
need to communicate with each other. These are completely normal on any 
Windows 10 machine.

The full port scan confirmed there are no services hiding on unusual 
ports and nothing unexpected was found beyond the standard Windows 
service footprint.

## Summary of Findings
| Port | Service | Risk Level | Notes |
|---|---|---|---|
| 135 | Microsoft RPC | Low | Standard Windows service |
| 139 | NetBIOS | Medium | Leaks hostname and network info |
| 445 | SMB | High | Historic attack vector, WannaCry used this |
| 5040 | Unknown | Low | Windows Device Association Framework |
| 7680 | Windows Update | Low | Normal Windows behavior |
| 49664-49670 | Dynamic RPC | Low | Standard Windows RPC range |

## What can an Attacker Does With This
With this information an attacker now knows the exact OS version, the 
hostname, that SMB is running and that no unusual security tools appear 
to be blocking connections. Port 445 would be the first thing a real 
attacker investigates for known vulnerabilities. In our case we are 
going to move forward with delivering a reverse shell payload instead 
of exploiting SMB directly.

## What is Next
With reconnaissance complete I now have a clear picture of the target. 
The next step is creating a malicious payload using msfvenom on Kali 
and setting up a listener in Metasploit to catch the reverse shell 
connection when the victim executes the file.

## Screenshots

aggressive scan output showing open ports and OS detection
  <img width="1074" height="793" alt="Screenshot 2026-03-12 160505" src="https://github.com/user-attachments/assets/1ee97bc5-912f-4194-ba9a-c457c881f5f0" />

  
full port scan output showing all 65535 ports scanned and additional services discovered
  <img width="952" height="442" alt="Screenshot 2026-03-12 161808" src="https://github.com/user-attachments/assets/5f454bcc-e7fe-4523-906e-dbbde157974b" />
  
