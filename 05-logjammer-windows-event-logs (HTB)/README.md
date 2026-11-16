# HTB Sherlock LogJammer - Reading Windows event logs without a SIEM

This time I didn't have Splunk. I got 5 raw .evtx files from a Windows workstation (DESKTOP-887GK2L) and had to find out what the user CyberJunkie did on it. I opened them in Event Viewer and filtered by Event ID. The user logged in, opened the firewall for Metasploit, created a scheduled task that runs a PowerShell script, downloaded SharpHound (Defender caught it), and in the end cleared the firewall log.

I'm still learning, so I wrote this the way I understood it.

## Links

- https://app.hackthebox.com/sherlocks/LogJammer

## Logs I had

- Security.evtx
- System.evtx
- Windows Firewall-Firewall.evtx
- Windows Defender Operational.evtx
- Windows PowerShell Operational.evtx

## What happened

CyberJunkie logged in on the machine itself (logon type 2). After that he added an outbound firewall rule called "Metasploit C2 Bypass", changed the audit policy, and created a scheduled task "HTB-AUTOMATION" that runs a .ps1 script from his Desktop. He also downloaded SharpHound, which is used to map Active Directory before an attack, but Defender put it in quarantine. He ran a PowerShell command to get the hash of his script. At the end he cleared the firewall log, probably to hide the rule he added.

## Step by step

Everything was on DESKTOP-887GK2L, 27 Mar 2023. Times below are UTC.

**1. When did he log in**
Security.evtx, filter Event ID 4624 (successful logon). The logon for CyberJunkie with Logon Type 2 (interactive) was at 14:37:09 UTC.
Event Viewer showed 18:37:09 because it converts to my PC time (Baku, UTC+4). I checked the real time in Details -> XML View, `TimeCreated SystemTime` ends with Z, which means UTC. I almost submitted the wrong time here.

**2. Firewall rule**
Windows Firewall log, Event ID 2004 (a rule was added). Rule name "Metasploit C2 Bypass", direction Outbound. Outbound means traffic from this PC to the internet, so this is for a payload to call back to a C2 server.

**3. Audit policy change (14:50:03)**
Security.evtx, Event ID 4719 (system audit policy was changed).
- Category: Object Access
- Subcategory: Other Object Access Events
- Change: Success added

My Event Viewer is in Russian, so the fields were in Russian. I used the subcategory GUID `{0cce9227-69ae-11d9-bed3-505054503030}` to make sure about the English name. This subcategory is what logs scheduled task creation (4698), so it connects to the next step.

**4. Scheduled task**
Security.evtx, Event ID 4698 (scheduled task created). The event has the task XML inside. The useful part was `<Actions>`:
- Task name: HTB-AUTOMATION
- Command: C:\Users\CyberJunkie\Desktop\Automation-HTB.ps1
- Arguments: -A cyberjunkie@hackthebox.eu

A task that runs a .ps1 from the Desktop is not normal.

**5. Defender caught SharpHound**
Defender Operational log:
- 1116 (threat detected): SharpHound, path C:\Users\CyberJunkie\Downloads\SharpHound-v1.1.0.zip
- 1117 (action taken): Quarantine

SharpHound collects users, groups and permissions from AD for BloodHound. So someone was preparing to look for a path to admin.

**6. PowerShell**
PowerShell Operational, Event ID 4104 (script block logging). The command was:
`Get-FileHash -Algorithm md5 .\Desktop\Automation-HTB.ps1`
He checked the hash of the same script the scheduled task runs.

**7. Log cleared**
System.evtx, Event ID 104 (log file was cleared). In Details the channel was `Microsoft-Windows-Windows Firewall With Advanced Security/Firewall`. So he cleared the firewall log, the same log where the Metasploit rule was. For the Security log it would be 1102, but here it was 104 because it's a different log.


## IOCs

- Host: DESKTOP-887GK2L
- Account: CyberJunkie
- Firewall rule: Metasploit C2 Bypass (Outbound)
- Scheduled task: HTB-AUTOMATION
- Script: C:\Users\CyberJunkie\Desktop\Automation-HTB.ps1
- Email in task arguments: cyberjunkie@hackthebox.eu
- Tool: C:\Users\CyberJunkie\Downloads\SharpHound-v1.1.0.zip

## MITRE ATT&CK

- T1078 - Valid Accounts
- T1562.004 - Disable or Modify System Firewall
- T1053.005 - Scheduled Task
- T1087.002 - Domain Account Discovery (SharpHound)
- T1059.001 - PowerShell
- T1070.001 - Clear Windows Event Logs

## What I would recommend

- Disable the CyberJunkie account
- Delete the "Metasploit C2 Bypass" rule and the HTB-AUTOMATION task
- Get Automation-HTB.ps1 and check what it does
- Check if anything else came from SharpHound (other tools, other hosts)
- Send these logs to a SIEM so clearing them on the host doesn't remove the evidence
- Alert on 1102, 104, 2004, 4698 and 4719


## Screenshots

- screenshots/01-logon-4624-utc.png - 4624 logon type 2, XML view with UTC time
- screenshots/02-firewall-rule-2004.png - Metasploit C2 Bypass rule, Outbound
- screenshots/03-audit-policy-4719.png - audit policy change, Other Object Access Events
- screenshots/04-scheduled-task-4698.png - HTB-AUTOMATION with the .ps1 command and arguments
- screenshots/05-defender-sharphound-1116.png - SharpHound detected
- screenshots/06-defender-quarantine-1117.png - action: Quarantine
- screenshots/07-powershell-4104.png - Get-FileHash command
- screenshots/08-log-cleared-104.png - firewall log cleared
