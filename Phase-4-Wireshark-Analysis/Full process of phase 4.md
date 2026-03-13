# Phase 4 — Wireshark Traffic Analysis

## What I'm Trying to Do
This phase is where I switch from attacker to defender mindset. Instead 
of launching the attack I am now analyzing what the attack looks like 
on the wire. The goal is to capture the reverse shell traffic live in 
Wireshark and identify exactly what a SOC analyst would see when 
investigating a compromised machine.

This is one of the most important skills in a SOC role. Tools like 
Splunk and Microsoft Sentinel ingest network traffic and generate alerts 
but understanding what the raw packets look like is what separates a 
good analyst from a great one in my opinion.

## Tool Used
Wireshark 4.6.4 running on Windows 10 victim machine. Capturing traffic 
from the victim side is more realistic because in a real investigation 
you are usually analyzing traffic logs from the compromised host, not 
the attacker.

## Setting Up the Capture
Wireshark was started on Windows 10 before the Metasploit listener was 
activated on Kali. The Ethernet adapter was selected for capture since 
that is the interface handling all traffic between the VMs on the 
LabNet-Attack internal network.

With Wireshark running I started the Metasploit listener on Kali and 
Free_VPN.exe which was still running in the background on Windows 10 
immediately connected back, generating live traffic for Wireshark to 
capture.

## Finding 1 — Malware Beaconing Behavior

Before the Metasploit listener was active I noticed something 
interesting in the capture. Wireshark was showing red and black packets 
on port 4444 even though no listener was running on Kali yet.

What was happening is that Free_VPN.exe was already running in the 
background and repeatedly trying to connect back to Kali on port 4444. 
Since no listener was active Kali was responding with RST ACK packets 
which means connection reset, essentially slamming the door shut every 
time the malware knocked.

This is called beaconing. Real malware is designed to keep trying to 
reach its command and control server even if the connection fails. It 
will keep attempting at regular intervals until it either succeeds or 
gets killed. In a real SOC environment this repeated outbound connection 
attempt to a non standard port from an unknown executable would 
immediately trigger an alert.

What the packets showed:
```
Black packets → RST ACK from Kali (port 4444 rejecting connections)
Red packets   → TCP port reuse errors from Free_VPN.exe retrying
ARP broadcast → Windows 10 normal network behavior mixed in
```

This screenshot alone tells a complete story of a compromised machine 
trying to phone home.

## Finding 2 — Active Meterpreter Session Traffic

Once the Metasploit listener was started on Kali the connection was 
established immediately. Applying the display filter:
```
tcp.port == 4444
```

This filtered out all other traffic and showed only the reverse shell 
communication. The traffic pattern changed completely from the failed 
beaconing to active sustained communication between the two machines.

To dig even deeper I applied a more specific filter to isolate only 
the packets carrying actual data:
```
tcp.port == 4444 && tcp.flags.push == 1
```

This filters out the acknowledgement packets and shows only the packets 
where data was actively being pushed through the connection. Each burst 
of PSH packets corresponds directly to a meterpreter command being 
executed and the results being sent back. This level of filtering is 
useful in a real investigation because it cuts through the noise and 
shows you exactly when the attacker was actively interacting with the 
victim machine versus when the connection was just sitting idle.

What the filtered packets showed:
```
192.168.1.18 → 192.168.1.17  PSH ACK  (victim sending data to Kali)
192.168.1.17 → 192.168.1.18  ACK      (Kali acknowledging receipt)
```

PSH ACK packets mean data is actively being pushed through the 
connection. Every command I ran in meterpreter like sysinfo, getuid, 
ps and ls generated a burst of these packets as the victim machine sent 
back the results.

## Finding 3 — Encrypted Payload Data

Clicking into any PSH ACK packet and expanding the Data section in the 
middle panel revealed the raw bytes being transmitted:
```
Data: 8d8181363541b79d3a6b0c8f723eba97eaa84239f8d8181628d81813b8d818163e942a351b27a62e9835cffd
Length: 112 bytes
```

The data is completely encrypted which is by design. Meterpreter 
encrypts all of its communication by default making it harder to detect 
through deep packet inspection. In a real SOC environment this is 
actually a red flag in itself because legitimate software running on 
non standard ports with fully encrypted traffic is highly suspicious 
behavior.

## What a SOC Analyst Would Flag

Looking at this capture from a defender perspective the following 
observations would trigger an investigation:

An unknown executable called Free_VPN.exe was making repeated outbound 
connection attempts to an internal IP on port 4444 which is not a 
standard service port. The connection was eventually established and 
sustained with encrypted data flowing in both directions. The beaconing 
behavior before the connection succeeded is consistent with known 
command and control malware patterns documented in the MITRE ATT&CK 
framework under T1071 Application Layer Protocol and T1571 Non-Standard 
Port.

## pcap File
The full packet capture was saved as reverse-shell-capture.pcap and is 
included in this repository. Anyone reviewing this project can open it 
in Wireshark and apply the tcp.port == 4444 filter to see exactly what 
was captured during the attack.

## Screenshots
- [Malware beaconing before listener was active showing RST ACK packets](screenshots/wireshark-beaconing.png)
- [Filtered view showing only port 4444 reverse shell traffic](screenshots/wireshark-filtered-4444.png)
- [Push data filter showing only active command traffic](screenshots/wireshark-push-data.png)
- [Active meterpreter session traffic with PSH ACK data packets](screenshots/wireshark-packet-data.png)
- [Colored traffic view showing purple TCP and yellow retransmissions](screenshots/wireshark-color-view.png)

