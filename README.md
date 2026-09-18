Tryhackme — Phishing Email Analysis Writeup
Platform: TryHackMe Room: The Greenholt Phish (https://tryhackme.com/room/phishingemails5fgjlzxc) Category: Phishing Analysis / Email Forensics / SOC Difficulty: Easy Date completed: 9-18-2026 Author: Chris Janciga

Overview

A sales executive at Greenholt PLC escalated a suspicious email to the SOC. The employee stated that this type of behavior does not align with the customers usual communication style and the message raised several red flags listed below. This writeup documents the investigation: extracting header artifacts, verifying the sender's authenticity through SPF/DMARC, and assessing the attachment for malicious content."

Reported red flags:

Generic greeting
Unexpected request for money transfer
Unsolicited attachment

Objective: Determine whether the email is legitimate or a phishing attempt, and document supporting evidence.

Tools Used/Can be Used
Tool	Purpose
Text editor / cat	Reading raw .eml headers
whois / ipinfo.io	IP ownership lookup
dig / MXToolbox	SPF and DMARC record retrieval
sha256sum	Hashing the attachment
VirusTotal	Reputation check + file-type identification
Investigation
Phase 1 — Header Artifact Extraction

Method: Opened challenge.eml in thunder and examined the raw headers.

Artifact	Header field	Value
Transfer Reference Number	Subject:	09674321
Sender display name	From:	Mr. James Jackson
Sender email address	From:	info@mutawamarine.com
Reply-to address	Reply-to: info.mutawamarine@mail.com

Analysis/RedFlags: Email is not professionally formatted / suspicious attachment file type / suspicious emails, not SWIFT domain emails


Phase 2 — Message Source & Origin

Explain: the Received: headers log each mail server hop. Reading them bottom-to-top traces the message back to its origin — the bottom-most Received line is the first hop, i.e. where it actually came from.

Originating IP address: 192.119.71.157 How you found it: In the email source file.

IP ownership (WHOIS):

Field	Value
IP address	192.119.71.157
Owner / Organization	HostPapa

Analysis: ip address can be found by look at the source of the email, and a whois who is 192.119.71.157 returns you with the owner.

Phase 3 — Email Authentication (SPF / DMARC)

SPF and DMARC are DNS-based mechanisms that let a domain declare who is allowed to send mail on its behalf and what to do with mail that fails. Checking them against the Return-Path domain tells you whether the sending server was authorized.

Return-Path domain identified: mutawamarine

SPF Record

Command / tool: dig txt mutawamarine (or MXToolbox SPF lookup)

Result: v=spf1 include:spf.protection.outlook.com -all


DMARC Record

Command / tool: dig txt _dmarc.mutawamarine.com (or MXToolbox DMARC lookup)

Result: v=DMARC1; p=quarantine; fo=1


Phase 4 — Attachment Analysis

attachments are the most common phishing payload. You extract the file, hash it to create an IOC, then check that hash against threat-intel sources without detonating the file.

Attachment filename: SWT_#09674321____PDF__.CAB Found in the email itself.

SHA256 hash:2e91c533615a9bb8929ac4bb76707b2444597ce063d84a4b33525e25074fff3f

VirusTotal findings:

Detection ratio	49/64
File size	400.26 KB (409868 total bytes)
Claimed extension	CAB
Actual file type	RAR

Analysis: The key finding here is often the mismatch between the claimed extension and the actual file type — e.g. a file presented as a document that is really an executable or script. It is very common for a file to be changed to something innocent looking like a .txt or .pdf but run a .exe in the background when you open them.

Indicators of Compromise (IOCs)

Type	Indicator
Sender address:	info@mutawamarine.com
Reply-To address:	info.mutawamarine@mail.com
Originating IP:	HostPapa
Attachment filename:	SWT_#09674321____PDF__.CAB
SHA256:	 2e91c533615a9bb8929ac4bb76707b2444597ce063d84a4b33525e25074fff3f

Conclusion: Phishing

Primary evidence: The email is an instant read flag to the employee because it is not how the customer usually communicates and was an unexpected email including asking for money which is a common which is a common tactic of phishing emails to impersonate someone you know for financial gain. The other major indicator of this email being illegitimate and a phishing attempt is the unsolicited attachment also attached to the email which presents itself as a .CAB file but is really a malicious .rar file that was detected on 49/64 virus total detectors.

Defensive Takeaways

User-reported escalation worked — the executive noticed the behavioral mismatch. Reinforces the value of SETA/awareness training.

Techniques Used:
Basic analysis of the email viewing the front end of the email as well as reviewing the source of the email to find IP addresses. Basic commands to pull SPF and DMARC records then analyzing the SHA256 hash in VirusTotal

Writeup by Chris Janciga · 9/18/26
