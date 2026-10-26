# SOC Simulator - Phishing that turned into data theft

My second scenario in the TryHackMe SOC Simulator. This one was a full attack from start to finish on a single host (win-3450, user michael.ascot). It started with a phishing email and ended with the attacker stealing financial files over DNS. The main thing I learned was how to connect separate alerts into one incident.

I'm still learning, so I wrote this the way I understood it.

## Links

- https://tryhackme.com/soc-sim/public-summary/2ca86ff07cbdf583fe33c17543a745416eb16fdaa435ef221f27c692d3ee7f5107ff00026dcfd3bbc1ba72fb6262855e

## What happened

An attacker emailed a fake invoice (.zip) to a VIP user, michael.ascot. He opened it. After that the attacker used PowerShell to drop a recon tool, got into a financial file share, copied the files, and sent them out over DNS to their own domain. All the attacker steps came from the same PowerShell process (PID 3728) - that's what helped me realize it was all one incident and not random events.


## Step by step

Everything was on host win-3450, user michael.ascot.

**1. The phishing email (1005)**
Subject "FINAL NOTICE: Overdue Payment", from john@hatmakereurope.xyz, attachment ImportantInvoice-Febrary.zip. It had everything a phishing email usually has: urgency, legal threats, a weird .xyz domain, and even a misspelled file name (Febrary). A .zip "invoice" is a common way to deliver malware.

**2. He actually opened it**
I searched the SIEM and saw Outlook create and open the .eml file. It also had a Zone.Identifier stream, which means Windows tagged it as downloaded from the internet. So I could prove the email was really opened, not just sitting in the inbox.

**3. Recon tool shows up (1020)**
A file called PowerView.ps1 appeared in the Downloads folder, created by powershell.exe. I looked it up - PowerView is a known hacker tool for Active Directory recon. A normal employee has no reason to have this. This is where I knew the host was compromised.

**4. Attacker goes for the files (1021)**
net.exe use Z: \\FILESRV-01\SSF-FinancialRecords - launched by the same PowerShell. The attacker mapped a drive right to the financial records share.

**5. Copying the data (1023)**
Robocopy.exe . C:\Users\michael.ascot\downloads\exfiltration /E - copied everything from the Z: drive into a folder literally named "exfiltration". That folder name made the intent pretty obvious.

**6. Cleaning up (1024)**
net.exe use Z: /delete - removed the drive mapping after copying, so there's less to notice.

**7. Sending the data out (1025-1032)**
A bunch of nslookup.exe queries to haz4rdw4re.io. The subdomain parts were Base64 text. This is DNS tunneling - the attacker turned the stolen files into DNS lookups to sneak the data out, because DNS usually isn't blocked.

## I decoded some of the DNS queries

The Base64 in the nslookup subdomains was not random. I decoded a few:
- UEsDBBQ... = PK (that's the ZIP file header, so the data was zipped first)
- ClientPortfolio
- Summary.xlsx
- InvestorPresentation2023.pptx

So I could actually see which files were being stolen, just from the DNS traffic. That was a cool moment for me.

## The lure script

In the attachments folder I found the script the attacker used (forceupdate.ps1). I opened it with Get-Content (read only, I did not run it). It pretends to be a Windows Update and diagnostics tool "crafted by Yani Zubair from IT" with a contact email, to make it look trustworthy. That fake IT author and the official-looking description are social engineering - it's meant to make the user feel safe running it.

## Trying to catch it in Wireshark

I tried to catch the DNS exfiltration in Wireshark from the analyst VM. I started a capture on Ethernet 3 and filtered for `dns.qry.name contains "haz4rdw4re"` but got nothing. When I filtered on just `dns`, I could see normal traffic (Windows and Microsoft stuff, Amazon, and our DNS server 10.0.0.2), but no queries to haz4rdw4re.io. That makes sense - the attack was happening on win-3450, not on my VM, and the two machines are on different parts of the network, so my VM never saw the victim's DNS traffic. So instead of the network capture, I investigated the exfiltration using the host logs in the SIEM (Sysmon Event ID 1), which showed the nslookup commands to haz4rdw4re.io.

## Kill chain

- Initial Access -> phishing email with .zip
- Execution -> user opened it, PowerShell ran
- Discovery -> PowerView for AD recon
- Collection -> mapped the share, Robocopy to local folder
- Defense Evasion -> deleted the drive mapping
- Exfiltration -> DNS tunneling to haz4rdw4re.io

## IOCs

- Sender: john@hatmakereurope.xyz
- Attachment: ImportantInvoice-Febrary.zip
- Exfil domain: haz4rdw4re.io
- Tool: PowerView.ps1
- Main process: powershell.exe (PID 3728) - parent of all the attacker steps
- Staging folder: C:\Users\michael.ascot\downloads\exfiltration
- Host: win-3450 / User: michael.ascot

## MITRE ATT&CK

- T1566.001 - Spearphishing Attachment
- T1204.002 - User Execution (malicious file)
- T1059.001 - PowerShell
- T1105 - Ingress Tool Transfer (PowerView)
- T1087 - Account Discovery
- T1021.002 - SMB / Windows Admin Shares
- T1074.001 - Local Data Staging (Robocopy)
- T1070 - Indicator Removal (drive unmap)
- T1048.003 - Exfiltration Over DNS
- T1071.004 - DNS

## What I would recommend

- Isolate win-3450 right away to stop the data leaving
- Block haz4rdw4re.io on the DNS firewall and the network firewall
- Reset michael.ascot's password
- Check FILESRV-01 logs to see exactly what was taken
- Keep the host and the exfiltration folder for forensics
- Treat this as a real data breach and tell management

## Screenshots

- images/01-powerview-dropped.png - PowerView.ps1 created in Downloads
- images/02-dns-exfil-nslookup.png - the nslookup DNS exfil queries to haz4rdw4re.io
- images/03-forceupdate-vm.png - the attacker's lure script (forceupdate.ps1) disguised as a Windows Update tool
- images/04-decoded-base64.png - decoded file names from the DNS queries (ClientPortfolio, Summary.xlsx)
- images/05-siem-splunk-logs.png - SIEM search on michael.ascot showing the chain (PowerView -> net use -> Robocopy)