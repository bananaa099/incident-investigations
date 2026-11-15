# THM Investigating Windows - A web server with backdoors everywhere

My first Windows host investigation without a SIEM. I got RDP access to a Windows Server 2016 machine (EC2AMAZ-I8UHO76) that was already hacked, and had to find out what the attacker did using only built-in Windows tools. It turned out the attacker didn't leave one backdoor, they left like six. The main thing I learned here was how many different places persistence can hide on Windows.

I'm still learning, so I wrote this the way I understood it.

## Links

- https://tryhackme.com/room/investigatingwindows

## What happened

The attacker got in through the website and uploaded a JSP web shell into the IIS folder. From there they ran Mimikatz to dump passwords, made a backdoor admin account (Jenny) and added the Guest account to Administrators. Then they set up persistence in a bunch of ways: a Run key that starts PsExec on every boot, scheduled tasks with fake system names, a PowerShell netcat listener, and a firewall rule for port 1337. On top of that they edited the hosts file to send google.com to their C2 server and to block antivirus and VirusTotal.

## Step by step

Everything was on EC2AMAZ-I8UHO76, compromise date 03/02/2019 (system time zone is UTC).

**1. What machine is this**
`systeminfo` - Windows Server 2016 Datacenter, build 14393, running on Amazon EC2.

**2. Who are the admins**
`net localgroup administrators` showed Administrator, Guest, Jenny and ssm-user. Guest should never be an admin, it's usually disabled. And `net user Jenny` said she never logged in at all. An admin account nobody ever used looks a lot like a backdoor account.

**3. When did people log in**
`net user John` - last logon 03/02/2019 5:48:32 PM. In Event Viewer (Security log) I filtered Event ID 4672 (special privileges assigned to new logon). The first one during the compromise was 03/02/2019 4:04:49 PM.

**4. What runs on startup**
`reg query HKLM\Software\Microsoft\Windows\CurrentVersion\Run` - there was an entry that starts `C:\TMP\p.exe` and connects to 10.34.2.3. When I logged in to the VM, a window was already open: "PsExec v1.98 ... Connecting to 10.34.2.3". So p.exe is just PsExec with a different name, and it runs commands on another host every time the server boots.

**5. Scheduled tasks**
In Task Scheduler I didn't trust the names and checked the Actions tab of every task. "Clean file system" sounds normal, but its action is `C:\TMP\nc.ps1 -l 1348` - a PowerShell netcat that listens on port 1348, every day at 4:55 PM. There was also "falshupdate22" (misspelled flash update, repeats every 2 minutes) and "check logged in". None of these are default Windows tasks.

**6. The C:\TMP folder**
This is where all the attacker tools were. Mimikatz was there with its output files, so the attacker dumped credentials from memory. `dir /T:C C:\TMP` showed the files were created on 03/02/2019, which is how I got the compromise date.

**7. The web shell**
In `C:\inetpub\wwwroot` there was a file with a .jsp extension. That's a web shell - the attacker uploaded it through the site and used it to run commands. This is most likely how they got in.

**8. The hosts file**
`notepad C:\Windows\System32\drivers\etc\hosts` - this was the part I liked the most:
- `76.32.97.132 google.com` - DNS poisoning, google.com goes to the attacker's C2
- `10.2.2.2 update.microsoft.com` - fake Windows Update server
- `127.0.0.1 www.virustotal.com` - blocks VirusTotal
- `127.0.0.1 dci.sophosupd.com` - blocks Sophos antivirus updates

So with one text file the attacker got C2 and also blinded the security tools.

**9. Firewall**
`wf.msc` -> Inbound Rules. The top rule was "Allow outside connections for development" and it opens TCP 1337. The name sounds like something a developer would add, but it was the attacker.

## Why removing one thing isn't enough

What surprised me in this case was how many persistence methods there were on one host: web shell, Run key, three scheduled tasks, two admin accounts, a firewall rule. If you only delete the scheduled task, the attacker is still in. That's why I think this server should be rebuilt from a clean image, not just cleaned.

## Kill chain

- Initial Access -> JSP web shell uploaded through the website
- Credential Access -> Mimikatz in C:\TMP
- Persistence -> Run key, scheduled tasks, backdoor admin Jenny, Guest added to admins
- Execution -> PowerShell netcat listener on 1348
- Lateral Movement -> PsExec to 10.34.2.3 on every boot
- Defense Evasion -> fake task names, hosts file blocks AV, firewall rule for 1337
- Command and Control -> google.com redirected to 76.32.97.132

## IOCs

- C2 IP: 76.32.97.132
- Fake update server: 10.2.2.2
- PsExec target: 10.34.2.3
- Files: C:\TMP\p.exe (renamed PsExec), C:\TMP\nc.ps1, Mimikatz in C:\TMP
- Web shell: .jsp file in C:\inetpub\wwwroot
- Ports: 1337 (firewall rule), 1348 (netcat listener)
- Accounts: Jenny (backdoor admin), Guest (added to Administrators)
- Scheduled tasks: Clean file system, falshupdate22, check logged in
- Firewall rule: Allow outside connections for development


## What I would recommend

- Isolate the server and rebuild it from a clean image
- Reset passwords for every account that ever logged in here (Mimikatz ran, so assume they're stolen)
- Delete Jenny, remove Guest from Administrators and disable it
- Block 76.32.97.132 and 10.2.2.2, and check 10.34.2.3 because PsExec was run against it
- Fix the file upload on the website and don't allow executable files in wwwroot
- Turn on LSA Protection / Credential Guard
- Send Security and Sysmon logs to a SIEM and alert on 4720, 4732, 4698, 7045 and 1102

## Screenshots

- screenshots/01-systeminfo.png - systeminfo output, Windows Server 2016
- screenshots/02-local-administrators.png - Guest and Jenny in local Administrators
- screenshots/03-run-key-psexec.png - PsExec (p.exe) connecting to 10.34.2.3 on startup
- screenshots/04-scheduled-task-nc-listener.png - "Clean file system" task running nc.ps1 -l 1348
- screenshots/05-hosts-file-poisoning.png - hosts file blocking VirusTotal and Sophos
- screenshots/06-firewall-rule-1337.png - inbound rule opening port 1337
