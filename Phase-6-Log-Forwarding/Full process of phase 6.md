# Phase 6  Centralized Log Forwarding with Windows Server 2022

## What I'm Trying to Do
In a real enterprise SOC environment no analyst sits at individual 
machines checking logs one by one. Every machine in the organization 
forwards its logs to a central collector and the SOC team monitors 
everything from one place.

This phase was about replicating that exact setup. The goal was to 
configure Windows 10 to forward its security event logs to Windows 
Server 2022 so that the attack artifacts we found in Phase 5 would 
also be visible on the server without having to log into the victim 
machine directly.

## Lab Environment for This Phase
```
Windows 10       -> 192.168.1.18  (log source, the victim machine)
Windows Server   -> 192.168.1.222 (log collector)
```

Both machines are on the same LabNet-Attack internal network and can 
communicate with each other as confirmed by successful ping tests at 
the start of this phase.

## Understanding Windows Event Forwarding

Windows Event Forwarding works using two main components:

The first is WinRM which stands for Windows Remote Management. This is 
the service that handles the actual network communication between 
machines. It needs to be running and properly configured on both the 
source machine and the collector before any logs can be forwarded.

The second is the Windows Event Collector service which runs on the 
server side and is responsible for receiving, storing and organizing 
the forwarded logs from source machines.

There are two forwarding modes. Collector Initiated means the server 
reaches out to the source machines and pulls the logs on a schedule. 
Source Computer Initiated means each machine pushes its own logs to 
the server. For this lab I attempted Collector Initiated first since 
it is the more common enterprise approach.

## Step 1  Configuring WinRM on Windows 10

Opened Command Prompt as Administrator on Windows 10 and ran:
```cmd
winrm quickconfig
```

The first attempt failed because the network profile on Windows 10 
was set to Public. WinRM requires the network to be set to Private 
or Domain before it will allow remote management connections. This 
is a Windows security measure to prevent remote access on untrusted 
networks.

Fixed this by running the following in PowerShell as Administrator:
```powershell
Set-NetConnectionProfile -NetworkCategory Private
```

After changing the network profile to Private, running winrm quickconfig 
again succeeded with the following confirmation:
```
WinRM has been updated for remote management
WinRM firewall exception enabled
Configured LocalAccountTokenFilterPolicy to grant administrative 
rights remotely to local users
```

Verified WinRM was listening correctly:
```cmd
winrm enumerate winrm/config/listener
```

Output confirmed:
```
Transport: HTTP
Port: 5985
Enabled: true
Listening on: 192.168.1.18
```

## Step 2  Configuring Windows Event Collector on Server 2022

On Windows Server 2022 Command Prompt as Administrator ran:
```cmd
winrm quickconfig
wecutil qc
```

WinRM was already running on the server. The wecutil qc command 
configured the Windows Event Collector service which is the component 
that receives and stores forwarded logs on the server side.

Also added Windows 10 as a trusted host on the server:
```cmd
winrm set winrm/config/client @{TrustedHosts="DESKTOP-2KU324L"}
```

This tells the server to trust connections coming from the Windows 10 
machine by hostname.

## Step 3  Creating the Log Forwarding Subscription

On Windows Server 2022 opened Event Viewer and navigated to 
Subscriptions. Created a new subscription with the following settings:
```
Subscription name: Win10-Security-Logs
Destination log:   Forwarded Events
Subscription type: Collector Initiated
Source computer:   DESKTOP-2KU324L
```

This subscription tells the server to reach out to Windows 10 and 
collect its Security event logs on an ongoing basis.

## The Problem  Network Path Not Found

When testing the subscription connection the following error appeared:
```
The network path was not found
```

This was unexpected since both machines could ping each other 
successfully and WinRM was confirmed running on both. After 
troubleshooting I discovered the root cause.

Windows Server 2022 is joined to an Active Directory domain called 
Cybermoaiadtraining.com while Windows 10 is in a standard workgroup 
called WORKGROUP. These are two completely different network 
authentication environments.

Collector Initiated log forwarding relies on Kerberos authentication 
which is an Active Directory protocol. When the server tried to 
authenticate to Windows 10 it attempted to use Kerberos but Windows 10 
had no idea what Kerberos was since it was not part of the domain. This 
caused the authentication to fail completely which is what produced the 
network path not found error.

## Attempted Fix  Joining Windows 10 to the Domain

The logical solution was to join Windows 10 to the Cybermoaiadtraining.com 
domain so both machines would be in the same authentication environment. 
On Windows 10 I navigated to:
```
This PC -> Properties -> Advanced System Settings -> Comptuter Name (Tab) -> Change -> Domain
```

Entered Cybermoaiadtraining.com as the domain name and attempted to 
join. The attempt failed with the following error:
```
An Active Directory Domain Controller for the domain 
Cybermoaiadtraining.com could not be contacted
```

The reason this failed is a DNS issue. When a machine tries to join a 
domain it first needs to find the domain controller through DNS. Both 
machines were configured to use 8.8.8.8 as their DNS server which is 
Google's public DNS. Google's DNS has no knowledge of a private lab 
domain called Cybermoaiadtraining.com so the lookup failed completely 
and the domain controller could never be contacted.

Changing the DNS on Windows 10 to point to the server at 192.168.1.222 
was attempted but the domain join still failed, indicating that the DNS 
service on Windows Server 2022 was not fully configured to serve the 
Cybermoaiadtraining.com zone to machines outside the domain.

## Why This Happened and What I Learned
This is actually interesting. In real enterprise 
environments getting log forwarding working correctly requires proper 
Active Directory infrastructure including correctly configured DNS, 
Group Policy and domain membership. When any one of these components 
is missing or misconfigured the whole chain breaks.

The specific lesson here is that both machines were using public DNS 
throughout this lab which worked fine for internet access and general 
name resolution but completely breaks down when private domain 
resolution is needed. A proper Active Directory lab setup requires 
the DNS server role to be configured on the domain controller and 
all machines pointing to it as their primary DNS before any domain 
operations can succeed.

## What Was Successfully Accomplished
Even though full log forwarding was not achieved the following was 
successfully configured:

WinRM was fully configured and verified on both Windows 10 and Windows 
Server 2022. The Windows Event Collector service was configured on the 
server. A subscription was created targeting Windows 10 as the log 
source. The root cause of the failure was identified and understood. 
A domain join was attempted and the exact failure point in the DNS 
resolution chain was identified.

## Future Improvement
The next step to complete this phase would be to properly configure 
the DNS service on Windows Server 2022 to serve the Cybermoaiadtraining.com 
zone to internal machines, point all machines to the server as their 
primary DNS and then join Windows 10 to the domain and retest the 
subscription. This would be covered in a dedicated Active Directory 
lab which is planned as the next project after this one.

## Screenshots
- [WinRM listener confirmed active on Windows 10 showing HTTP transport 
on port 5985 and listening on 192.168.1.18](screenshots/winrm-configured.png)
- [WinRM successfully configured on Windows 10 for remote management 
with firewall exception enabled and admin rights granted](screenshots/winrm-win10-configured.png)
- [Windows Event Collector service configured on Server 2022 using 
wecutil qc confirming the server is ready to receive forwarded logs](screenshots/wecutil-configured.png)
- [Windows Server 2022 trusted hosts updated to DESKTOP-2KU324L 
allowing the server to authenticate connections from Windows 10](screenshots/server-trusted-hosts.png)
- [Event Viewer subscription created on Server 2022 with Windows 10 
added as the source computer for Security log collection](screenshots/subscription-computer-added.png)
- [Network path not found error when testing the subscription connection 
confirming the authentication failure between the domain joined server 
and the workgroup Windows 10 machine](screenshots/network-path-error.png)
- [Domain join attempt showing Active Directory domain controller 
could not be contacted due to DNS misconfiguration](screenshots/domain-join-failed.png)
  [Network path not found error when testing the subscription connection 
confirming the authentication failure between the domain joined server 
and the workgroup Windows 10 machine](screenshots/network-path-error.png)
