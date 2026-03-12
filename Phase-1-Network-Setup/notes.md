# Phase 1 - Network Setup

## What I'm Trying to Do
Before touching any attack tools or malware, I needed to build a safe 
environment to work in. The whole point of this phase was to get all 
four VMs talking to each other while making sure none of them can reach 
my real network or the internet. You don't want actual malware escaping 
into your home network.

## My Lab Machines
| VM | What It's For | IP Address | Network |
|---|---|---|---|
| Kali Linux | This is my attacker machine | 192.168.20.11 | LabNet-Attack |
| Windows 10 | This is my victim machine | 192.168.20.10 | LabNet-Attack |
| Windows Server 2022 | Log collection and analysis | 192.168.10.5 | LabNet-Corp |
| Metasploitable 2 | Extra vulnerable target for practice | 192.168.10.20 | LabNet-Corp |

## Why Internal Network?
VirtualBox gives you a few network options and I specifically chose 
Internal Network for this lab. Here's my thinking:

- **NAT** gives internet access but machines can't talk to each other 
  easily -- not useful for attack simulation
- **Bridged** puts your VMs on the same network as your real computer -- 
  way too risky when running malware
- **Host-Only** is isolated but your VMs still have a connection to the 
  host machine
- **Internal Network** is the sweet spot -- VMs talk to each other, 
  nothing gets out, host machine is completely untouched

For a malware lab this was the only real choice.

## Preparing the Machines Before Isolating Them
Before switching everything to Internal Network I made sure to update 
all machines while they still had internet access. Once you cut internet 
you can't get it back without changing the network settings again, so 
it made sense to do all updates first.

I also installed Wireshark on Windows 10 at this stage since I'll need 
it later to capture the reverse shell traffic live.

## Tools I Verified on Kali
Ran version checks on all the key tools before isolating the machine:

| Tool | Version |
|---|---|
| Metasploit Framework | 6.4.116-dev |
| Nmap | 7.98 |
| msfvenom | Included in Metasploit |

Everything came back clean. msfvenom shows "version unknown" which is 
normal — it's bundled inside Metasploit so it doesn't have its own 
version number.

## Software Installed
- **Wireshark 4.6.4** on Windows 10 - installed with Npcap enabled 
  which is required for live packet capture. Without Npcap, Wireshark 
  can see traffic but can't capture it.

## Snapshots Before Changing Anything
I took a clean snapshot of every machine before touching network 
settings. This way if anything breaks during configuration I can roll 
back instantly without reinstalling anything.

| VM | Snapshot Name |
|---|---|
| Kali Linux | Kali-Clean-Updated |
| Windows 10 | Win10-Clean-Updated |
| Windows Server 2022 | Server2022-Clean-Updated |
| Metasploitable 2 | Metasploitable2-Clean |

## Screenshots
<img width="1113" height="742" alt="Screenshot 2026-03-11 215940" src="https://github.com/user-attachments/assets/423f7bc8-b577-4841-8502-15d266e3df68" />
 above png version check output for Metasploit, 
  Nmap and msfvenom in Kali terminal
<img width="1008" height="759" alt="Screenshot 2026-03-11 215824" src="https://github.com/user-attachments/assets/c1c6cbdf-d4be-4e8d-b356-a8279cbc10ae" />
  above png is Wireshark open on Windows 10 showing 
  network adapters detected, confirms Npcap installed correctly
<img width="1017" height="749" alt="Screenshot 2026-03-11 232850" src="https://github.com/user-attachments/assets/dbdc505a-d98f-41fa-935a-73d9f6031372" />
  above png is Windows Server 2022 showing all updates installed

## What's Next
Once all snapshots are confirmed I'll switch all VMs to Internal Network 
mode in VirtualBox and assign static IPs to each machine. Then I'll 
verify connectivity by pinging between them before moving on to 
reconnaissance.

