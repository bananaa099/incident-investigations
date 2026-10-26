# SOC Simulator: Phishing

My first scenario in the TryHackMe SOC Simulator. I took alerts from the queue, checked them, and wrote a short report for each.

## Links
- https://tryhackme.com/soc-sim/public-summary/aa87cb941e4f8cd43cff47a1fd6214b3076403bc47d65cc2fbb2b1f44b68bde7d6729a17e32fa4ef51437f633fda4ecc

## Results

| ID | What it was | Verdict | Escalated |
|---|---|---|---|
| 8818 | Onboarding email to a new employee | False Positive | No |
| 8815 | Fake Amazon email | True Positive | No |
| 8816 | User clicked the Amazon link, firewall blocked it | True Positive | No |
| 8817 | Fake Microsoft email, user opened the login page | True Positive | Yes |

## 8818 - Onboarding email

j.garcia got an email from onboarding@hrconnex.thm asking her to finish her profile.
The link was clean in TryDetectThis, no attachment, nothing strange in the firewall logs.

Then I searched `hrconnex.thm` in the SIEM and found an internal email from h.harris to the IT team. It says hrconnex.thm is our new HR partner and that J. Garcia hadn't received her onboarding email. So the company was expecting this email.

Closed as False Positive.

## 8815 and 8816 - Fake Amazon email

h.harris got a "your package couldn't be delivered" email from urgents@amazon.biz with a bit.ly link.

Why I think it's phishing:
- amazon.biz is not Amazon
- link hidden with bit.ly
- TryDetectThis marked the link as MALICIOUS
- "reply in 48 hours" pressure

A minute later the user clicked the link, but the firewall blocked it (alert 8816).
True Positive, but I didn't escalate because the connection didn't go through.

## 8817 - Fake Microsoft email

c.allen got an "unusual sign-in" email from no-reply@m1crosoftsupport.co (1 instead of i).
The link went to a fake login page.

I searched the domain in the SIEM and saw that one minute after the email arrived, the user opened this page and the connection was **allowed**. So they could have typed their password.

I escalated to L2 and recommended:
- reset the user's password
- check sign-in logs
- block the domain and IP 45.148.10.131
- remove the email from all mailboxes

## What I learned

- Always check the firewall logs. Blocked or allowed decides if I escalate.
- True Positive doesn't always mean escalation.
- Look at the domain carefully: m1crosoft vs microsoft.
- Not every email with a link is phishing. Searching the SIEM for context helped me close 8818.

Note: SIEM timestamps are shown in a different time zone than the alerts.

## Screenshots

- `images/01-case-report-8817.png` - report for the Microsoft email
- `images/02-case-report-8816.png` - report for the blocked click
- `images/03-case-report-8818.png` - report for the false positive
- `images/04-siem-hrconnex-internal-email.png` - SIEM: hrconnex.thm is our HR partner
- `images/05-trydetectthis-bitly-malicious.png` - bit.ly link flagged as malicious
- `images/06-siem-m1crosoft-allowed.png` - SIEM: the email and the allowed connection one minute later