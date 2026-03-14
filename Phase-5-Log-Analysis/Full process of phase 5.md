# Phase 5  Windows Event Log Analysis

## Quick Note Before We Start

One thing I forgot to mention in the earlier phases the Windows 10 
victim machine username is wsp vro. Yes that is a real username on a 
real machine, judge me later. You will see it pop up throughout the 
logs and screenshots in this phase and going forward. Just know that 
wsp vro is the unsuspecting victim who kept downloading and running 
suspicious VPN software from random web servers. Some people never 
learn (thats ME I assume 😂).

## What I'm Trying to Do
This phase is about finding the forensic footprints left behind by the 
attack on the Windows 10 machine. While Wireshark showed what happened 
on the network, Event Viewer shows what happened inside the operating 
system itself. Together they build a complete picture of the attack from 
two different angles which is exactly how a real SOC analyst approaches 
an incident investigation.

## First Attempt  Audit Policy Not Enabled

When I first opened Event Viewer and searched for Event ID 4688 process 
creation events related to Free_VPN.exe, nothing came up. After some 
investigation I realized that Windows 10 does not enable process 
creation auditing by default. This means the operating system was not 
logging which processes were being launched so there was no record of 
Free_VPN.exe executing even though it clearly ran.

This is actually a very realistic and important finding. In real 
enterprise environments this is a common security gap. If audit policies 
are not configured before an incident happens the logs will not contain 
the evidence needed for investigation. A SOC team needs to make sure 
these policies are enabled proactively, not reactively.

## Enabling Audit Policies

Before rerunning the attack I enabled the necessary audit policies on 
Windows 10 so the logs would capture everything this time.

### Enabling Process Tracking in Local Security Policy

Opened Local Security Policy via secpol.msc and navigated to:
```
Security Settings → Local Policies → Audit Policy → Audit Process Tracking
```
Enabled both Success and Failure auditing. This tells Windows to log 
every process that launches or attempts to launch on the system.

### Enabling Command Line Logging

Navigated to:
```
Security Settings → Advanced Audit Policy Configuration → 
System Audit Policies → Detailed Tracking → Audit Process Creation
```
Enabled Success auditing. This extends the process creation logs to 
include the full command line used to launch each process, making it 
much easier to identify malicious activity.

Also enabled command line logging via the registry:
```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit
ProcessCreationIncludeCmdLine_Enabled = 1
```

Without this registry key Windows logs that a process ran but does not 
record what command was used to launch it. With it enabled the full 
command line is captured which is invaluable for forensic investigation.

## Rerunning the Attack

With audit policies now properly configured I restarted the Metasploit 
listener on Kali and Free_VPN.exe which was still running in the 
background on Windows 10 immediately reconnected. The session was 
established instantly since the payload never stopped trying to beacon 
home.

Since I had a live meterpreter session and the logging was now active 
I decided to make the most of it. I used the shell command in 
meterpreter to drop into a real Windows command prompt on the victim 
machine and ran several commands to simulate what a real attacker would 
do after gaining access:
```
shell     → dropped into Windows cmd.exe on victim machine
whoami    → confirmed which user account the malware was running as
ipconfig  → gathered network configuration information
net user  → enumerated user accounts on the machine
```

Each of these commands represents a real post exploitation technique. 
After gaining initial access a real attacker would immediately run 
these kinds of commands to understand the environment they are in, 
what privileges they have and what other targets might be accessible.

## Hunting for Evidence in Event Viewer

After running the commands I went back to Event Viewer on Windows 10 
and this time the logs told the complete story.

### Finding 1  Free_VPN.exe Process Creation (Event ID 4688)

Filtered Security logs by Event ID 4688 and searched for Free_VPN. 
This time it appeared immediately. The log entry contained:
```
Event ID:          4688
SubjectUserName:   wsp vro
NewProcessName:    C:\Users\wsp vro\Downloads\Free_VPN.exe
CommandLine:       "C:\Users\wsp vro\Downloads\Free_VPN.exe"
ParentProcessName: C:\Windows\explorer.exe
NewProcessId:      0x1818
Date:              3/14/2026 1:04 AM
```

This log entry proves exactly when the malware was executed, which user 
ran it, where it was located on disk and that it was launched by 
explorer.exe meaning the user double clicked it. This is a complete 
forensic record of the initial infection event.

### Finding 2  Attacker Shell Commands (Event ID 4688)

Searching further I found log entries for the commands run through the 
meterpreter shell. The most significant was whoami:
```
Event ID:          4688
NewProcessName:    C:\Windows\System32\whoami.exe
CommandLine:       whoami
ParentProcessName: C:\Windows\System32\cmd.exe
SubjectUserName:   wsp vro
Date:              3/14/2026 1:07:32 AM
```

This is the most powerful finding in the entire lab. The parent process 
being cmd.exe directly proves that a command shell was active on the 
victim machine and the attacker was running commands through it. The 
chain is clear:
```
Free_VPN.exe executed
         ↓
Meterpreter session established
         ↓
Shell command opened cmd.exe
         ↓
whoami executed through cmd.exe
         ↓
All of it logged by Windows Event Viewer
```

## Key Lesson Learned

The most important takeaway from this phase is that audit policies must 
be configured before an incident happens. The first time I looked for 
evidence it was not there because logging was not enabled. In a real 
SOC environment this would mean a compromised machine with no host 
based evidence to investigate.

Enabling process creation auditing and command line logging should be 
part of every organization's baseline security configuration. Without 
it a SOC team is essentially investigating blind.

## Screenshots
- [Local Security Policy showing audit process tracking enabled](screenshots/audit-policy-enabled.png)
- [Advanced audit policy with process creation logging enabled](screenshots/audit-process-creation.png)
- [Registry key enabling command line logging in process creation events](screenshots/cmdline-logging-enabled.png)
- [Event ID 4688 showing Free_VPN.exe execution with full details](screenshots/event-4688-freevpn.png)
- [Event ID 4688 showing whoami executed through cmd.exe by meterpreter](screenshots/event-4688-whoami.png)

