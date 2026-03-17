# Phase 7 Splunk SIEM Integration and Attack Detection

## What I'm Trying to Do
Phase 6 proved that Windows Event Forwarding was not going to work 
in this lab environment due to the domain and workgroup authentication 
gap between Windows 10 and Windows Server 2022. Rather than continuing 
to fight with a native Windows solution that was designed for fully 
configured Active Directory environments, I pivoted to using 
Splunk Enterprise as the SIEM on Windows Server 2022.

Organizations do not rely on Windows Event Forwarding as their primary 
log collection method. They use dedicated SIEM platforms like Splunk 
because they are more reliable, more powerful and give analysts a 
proper interface to search, correlate and visualize security events 
across the entire environment.

The goal of this phase was to get Windows 10 security logs flowing 
into Splunk and then use Splunk to detect the exact attack artifacts 
we found manually in Phase 5, this time through a proper SIEM 
detection workflow.

## Lab Architecture for This Phase
```
Windows 10 (Victim Machine)
Splunk Universal Forwarder installed
Monitors Windows Security Event Logs
        ↓
        TCP Port 9997
        ↓
Windows Server 2022 (SIEM)
Splunk Enterprise installed
Receives, indexes and searches logs
        ↓
SOC Detection Dashboard
```

This is the proper way to collect and analyze logs at scale. 
No analyst logs into individual machines to check Event 
Viewer. Everything flows into the SIEM and the SOC team monitors from 
one central place.

## Step 1  Installing Splunk Enterprise on Windows Server 2022

Before installing Splunk I had to temporarily switch Windows Server 
2022 from Internal Network back to NAT in VirtualBox to get internet 
access for the download. Once the download was complete I switched it 
back to Internal Network and reassigned the static IP 192.168.1.222.

Splunk Enterprise 10.2.1 was downloaded from splunk.com and installed. 
During installation I created an admin 
account with a strong password. After installation Splunk automatically 
started and was accessible through the browser at:
```
http://127.0.0.1:8000
```

The first thing I configured after logging in was the receiving port. 
This tells Splunk to listen for incoming logs from forwarders:
```
Settings -> Forwarding and Receiving -> Configure Receiving -> Port 9997
```

Port 9997 is Splunk's dedicated log receiving port. To make it easier
Think of it as a 
mailbox slot that the Universal Forwarder on Windows 10 will deposit 
logs into.

## Step 2 Installing Splunk Universal Forwarder on Windows 10

The Universal Forwarder is a lightweight agent that runs in the 
background on Windows 10 and forwards logs to the Splunk server. It 
does not have a graphical interface. It is purely a background service 
that collects and ships logs.

During installation I configured:
```
Deployment Server: 192.168.1.222 port 8089
Receiving Indexer:  192.168.1.222 port 9997
```

This told the forwarder exactly where to send logs the moment it 
started running.

## Step 3 Troubleshooting the Forwarder

This phase had a significant troubleshooting component that is worth 
documenting because I believe it reflects real world SIEM deployment challenges.

After installation the forwarder was not starting correctly due to an 
SSL certificate generation failure. The error was:
```
Unable to generate certificate for SSL. SSL certificate generation failed.
```

After researching the issue I tried several approaches including 
disabling SSL in the configuration files and reinstalling the forwarder 
with different settings. The SSL issue was caused by a conflict between 
the forwarder version and the Windows 10 environment.

The fix that worked was uninstalling the forwarder completely and 
reinstalling it, this time entering the deployment server and receiving 
indexer details during the installation wizard itself rather than 
configuring them afterward. This allowed the forwarder to initialize 
correctly and start without SSL errors.

After reinstalling confirmed the forwarder was running:
```
cd "C:\Program Files\SplunkUniversalForwarder\bin"
splunk status
Output: The Splunk daemon (splunkd) is already running
```

## Step 4 Configuring Log Collection

With the forwarder running I needed to tell it which logs to collect 
and forward to Splunk. This is done through a configuration file called 
inputs.conf located at:
```
C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf
```

The file did not exist by default so I created it using PowerShell:
```powershell
$path = "C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf"

@"
[default]
host = DESKTOP-2KU324L

[WinEventLog://Application]
disabled = 0

[WinEventLog://System]
disabled = 0

[WinEventLog://Security]
disabled = 0
"@ | Set-Content -Path $path -Force
```

This configuration tells the forwarder to monitor three Windows event 
log channels using the Windows Event Log API. The Security channel is 
the most important one for this lab because it contains the process 
creation events and logon events we care about.

An important lesson here is the difference between two methods of 
collecting Windows logs in Splunk:

The first method monitors the raw .evtx file directly from disk. This 
works but Splunk cannot properly parse the event fields which makes 
searching and detection much harder.

The second method uses WinEventLog:// inputs which connects to the 
Windows Event Log API directly. This is the correct method because 
it allows Splunk to extract structured fields like EventCode, 
Account_Name, Process_Name and ParentProcess which are essential for 
SOC investigations.

After creating inputs.conf I restarted the forwarder:
```
splunk restart
```

Then verified the connection between Windows 10 and the Splunk server 
by running netstat on Windows 10:
```
netstat -an | findstr 9997
```

Output confirmed an ESTABLISHED connection to 192.168.1.222:9997 
proving logs were flowing to the server.

## Step 5 Detecting the Attack in Splunk

With logs flowing into Splunk I ran targeted searches to find the 
attack artifacts from Phase 3 and Phase 5.

### Detection 1 Free_VPN.exe Execution
```
index=* host=DESKTOP-2KU324L EventCode=4688 Free_VPN
| table _time, New_Process_Name, Creator_Process_Name, Account_Name
```

This search returned 3 events showing:
```
Time                    New Process              Creator Process        User
2026-03-14 01:03:38    Free_VPN.exe             explorer.exe           wsp vro
2026-03-14 01:04:11    Free_VPN.exe             explorer.exe           wsp vro
2026-03-14 01:06:51    cmd.exe                  Free_VPN.exe           wsp vro
```

This table tells the complete infection story. The first two rows show 
Free_VPN.exe being launched by explorer.exe which means the user double 
clicked it. The third row shows cmd.exe being spawned by Free_VPN.exe 
which is the meterpreter shell opening a command prompt on the victim 
machine.

### Detection 2 Post Exploitation Commands
```
index=* host=DESKTOP-2KU324L EventCode=4688 whoami
| table _time, New_Process_Name, Creator_Process_Name, Account_Name
```

This returned 1 event:
```
Time                    New Process          Creator Process      User
2026-03-14 01:07:32    whoami.exe           cmd.exe              wsp vro
```

This confirms the attacker ran whoami through the meterpreter shell 
to identify the compromised user account. The parent process being 
cmd.exe directly ties this back to the shell that Free_VPN.exe opened.

## Step 6 Building the SOC Detection Dashboard

After confirming the searches worked I built a Splunk dashboard called 
SOC Attack Detection Lab (Moaiad Eljack) to visualize the attack artifacts in one place.

The dashboard contains three panels:

The first panel titled Malware Execution shows every instance of 
Free_VPN.exe launching and the cmd.exe shell it spawned, giving a 
clear picture of the initial infection and shell establishment.

The second panel titled Attacker Shell Commands shows the whoami 
command executed through the meterpreter shell, proving active 
attacker interaction with the victim machine.

The third panel titled All Process Creation Events shows a complete 
timeline of every process that ran on the victim machine during the 
attack window, giving full visibility into attacker activity.

Instead of manually clicking through Event Viewer one event at a time, the 
dashboard presents the entire attack timeline.

## Connecting This Back to the Full Lab

This phase completes the full SOC analyst workflow that this lab was 
built to demonstrate:
```
Phase 2  → Reconnaissance (attacker maps the target)
Phase 3  → Payload delivery (attacker gains access)
Phase 4  → Wireshark captures the network evidence
Phase 5  → Event Viewer finds the host evidence manually
Phase 6  → WEF fails, Splunk identified as the solution
Phase 7  → Splunk ingests logs and detects the attack automatically
```

The same Free_VPN.exe artifact that required manual hunting in Event 
Viewer in Phase 5 was detected automatically in Splunk in Phase 7 with 
a single search query. This is the difference between manual forensics 
and SIEM based detection at scale.

## Lessons Learned

The inputs.conf configuration file is critical for proper log 
ingestion. Without it the Universal Forwarder has no instructions on 
what to collect and Splunk receives nothing regardless of how well the 
network connection is configured.

Using WinEventLog:// inputs instead of raw .evtx file monitoring is 
the correct approach because it enables proper field 
extraction which makes Splunk searches and detections significantly 
more powerful and accurate.

Splunk search syntax uses the pipe character to chain commands 
together. The first part of the search filters the events and 
everything after the pipe formats or analyzes them.

## Splunk Detection Queries Used
```
Free_VPN.exe execution detection:
index=* host=DESKTOP-2KU324L EventCode=4688 Free_VPN
| table _time, New_Process_Name, Creator_Process_Name, Account_Name

Post exploitation command detection:
index=* host=DESKTOP-2KU324L EventCode=4688 whoami
| table _time, New_Process_Name, Creator_Process_Name, Account_Name

Full process timeline:
index=* host=DESKTOP-2KU324L EventCode=4688
| table _time, New_Process_Name, Creator_Process_Name, Account_Name
| sort _time
```

## Screenshots
- [Splunk Enterprise dashboard home after installation](screenshots/splunk-dashboard.png)
- [Universal Forwarder running confirmed on Windows 10](screenshots/forwarder-running.png)
- [Security log monitor added to Universal Forwarder](screenshots/monitor-added.png)
- [PowerShell command creating inputs.conf with WinEventLog inputs](screenshots/inputs-conf-created.png)
- [Splunk search detecting Free_VPN.exe execution via EventCode 4688](screenshots/splunk-freevpn-detected.png)
- [Splunk search detecting whoami command executed through meterpreter shell](screenshots/splunk-whoami-detected.png)
- [Statistics table panel showing malware execution chain](screenshots/splunk-table-panel.png)
- [Statistics table panel showing whoami post exploitation command](screenshots/splunk-whoami-panel.png)
- [All process creation events panel showing complete attack timeline](screenshots/splunk-all-processes-panel.png)
- [Final SOC Attack Detection Lab dashboard with all panels populated](screenshots/splunk-dashboard-final.png)
