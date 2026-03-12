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
| Kali Linux | This is my attacker machine | 192.168.1.17 | LabNet-Attack |
| Windows 10 | This is my victim machine | 192.168.1.18 | LabNet-Attack |
| Windows Server 2022 | Log collection and analysis | 192.168.1.222 | LabNet-Attack |
| Metasploitable 2 | Extra vulnerable target for practice | 192.168.1.21 | LabNet-Attack |

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

## Assigning Static IPs
Since Internal Network has no DHCP server, both VMs came up with no IP 
assigned. Windows 10 showed a self assigned 169.254.x.x address which 
is what Windows does when it cannot find a DHCP server, basically 
useless for lab work.

I manually assigned static IPs to both machines so they have permanent 
addresses that will not change between reboots. This is important 
because Metasploit listeners and Nmap scans need to target a specific 
IP. If it changes every reboot the whole lab breaks.

Kali Linux was assigned via Network Manager GUI:
- IP Address: 192.168.1.17
- Subnet Mask: 255.255.255.0
- Gateway: 192.168.1.1
- DNS: 8.8.8.8

Windows 10 was assigned via TCP/IPv4 Properties:
- IP Address: 192.168.1.18
- Subnet Mask: 255.255.255.0
- Gateway: 192.168.1.1
- DNS: 8.8.8.8

## Connectivity Testing
After assigning IPs I ran ping tests between both machines to confirm 
they could communicate.

First ping was from Kali to Windows 10 and it failed completely. 4 
packets were sent and 0 were received, 100% packet loss. After some 
troubleshooting I realized Windows Firewall was blocking inbound ICMP 
traffic which is the protocol ping uses. This is actually a really 
common thing you run into in real environments. Windows blocks ping by 
default as a security measure.

I disabled Windows Defender Firewall on all three network profiles 
(Domain, Private and Public) on the Windows 10 VM. For this lab we need 
full communication between machines without firewall interference since 
we are simulating an attack scenario.

After disabling the firewall the ping from Kali to Windows 10 was 
immediately successful. I then pinged in the opposite direction from 
Windows 10 to Kali and that worked as well. Both directions confirmed 
which means the network is fully operational and both machines can 
communicate freely.

## Connectivity Test Results
| Test | Result |
|---|---|
| Kali to Windows 10 ping | Failed initially, firewall was blocking |
| Kali to Windows 10 ping after firewall disabled | Successful |
| Windows 10 to Kali ping | Successful |

## Snapshots Taken After Network Configuration
| VM | Snapshot Name |
|---|---|
| Kali Linux | Kali-Network-Configured |
| Windows 10 | Win10-Network-Configured |

## Screenshots
 Below png's - ip a output in Kali terminal confirming 192.168.1.17 assigned
<img width="897" height="699" alt="Screenshot 2026-03-12 002506" src="https://github.com/user-attachments/assets/80e65745-1011-43d8-b87c-b1d3bb5fd56f" />
<img width="1026" height="612" alt="Screenshot 2026-03-12 002853" src="https://github.com/user-attachments/assets/4055bf93-303f-4b20-8e41-9949ca4477fd" />

  Below png's - ipconfig output in Windows 10 confirming 192.168.1.18 assigned
<img width="961" height="741" alt="Screenshot 2026-03-12 000546" src="https://github.com/user-attachments/assets/0d0375a2-54bb-4cc9-989e-cf6d98aa6601" />
<img width="1003" height="783" alt="Screenshot 2026-03-12 000629" src="https://github.com/user-attachments/assets/24999eaa-3b6e-4fbf-838c-7aa17c0c5c84" />

  Below png - ping from Kali to Windows 10 showing 100% packet loss due to firewall  
<img width="1044" height="614" alt="Screenshot 2026-03-12 003149" src="https://github.com/user-attachments/assets/8941e5d0-c385-40e9-9f3b-cdef28fb5dd1" />

  Below png - ping from Kali to Windows 10 after firewall disabled, all packets received
<img width="1025" height="704" alt="Screenshot 2026-03-12 003558" src="https://github.com/user-attachments/assets/8407db4f-487a-43ae-bf9e-a68830e18eb4" />

  Below png - ping from Windows 10 to Kali confirming both directions work
<img width="982" height="694" alt="Screenshot 2026-03-12 003741" src="https://github.com/user-attachments/assets/45843b29-757c-47c5-b6c9-f765326020e5" />
  

## What is Next
With the network fully configured and both machines communicating I am 
ready to move into Phase 2 which is reconnaissance. I will run Nmap 
scans from Kali against the Windows 10 machine to discover open ports 
and services, just like a real attacker would before launching an attack.
