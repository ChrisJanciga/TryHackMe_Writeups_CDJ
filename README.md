# TryHackme_TheGreenholtPhish_writeup
[Room Name] — Phishing Email Analysis Writeup

Platform: TryHackMe Room: [Room name + link] Category: Phishing Analysis / Email Forensics / SOC Difficulty: [Easy / Medium / Hard] Date completed: [YYYY-MM-DD] Author: Chris Janciga

Overview

A few sentences framing the scenario in your own words. What was reported, by whom, and what you were asked to determine. Example structure: "A sales executive at [company] escalated a suspicious email to the SOC. The message showed several red flags — [list them briefly]. This writeup documents the investigation: extracting header artifacts, verifying the sender's authenticity through SPF/DMARC, and assessing the attachment for malicious content."

Reported red flags:

[e.g. generic greeting]
[e.g. unexpected money-transfer request]
[e.g. unsolicited attachment]

Objective: Determine whether the email is legitimate or a phishing attempt, and document supporting evidence.

Tools Used
Tool	Purpose
Text editor / cat	Reading raw .eml headers
whois / ipinfo.io	IP ownership lookup
dig / MXToolbox	SPF and DMARC record retrieval
sha256sum	Hashing the attachment
VirusTotal	Reputation check + file-type identification
[add any others you used]	
Investigation
Phase 1 — Header Artifact Extraction

Briefly explain: email headers record the message's metadata and path. Opening the raw .eml in a text editor exposes fields the mail client normally hides.

Method: Opened challenge.eml in [tool] and examined the raw headers.

Artifact	Header field	Value
Transfer Reference Number	Subject:	[fill in]
Sender display name	From:	[fill in]
Sender email address	From:	[fill in]
Reply-to address	Reply-To:	[fill in]

Analysis: Note anything suspicious here. Does the display name match the actual sending address? Does the Reply-To differ from the From address — meaning replies would route somewhere other than the apparent sender? A mismatch is a classic phishing indicator; explain what you see.

Phase 2 — Message Source & Origin

Explain: the Received: headers log each mail server hop. Reading them bottom-to-top traces the message back to its origin — the bottom-most Received line is the first hop, i.e. where it actually came from.

Originating IP address: [fill in] How you found it: [which Received header].

IP ownership (WHOIS):

Field	Value
IP address	[fill in]
Owner / Organization	[fill in]
[Country / ASN, optional]	[fill in]

Analysis: Does the IP owner align with the domain the email claims to be from? A legitimate email from [customer domain] would typically originate from that organization's mail infrastructure. Explain whether this checks out or raises suspicion.

Phase 3 — Email Authentication (SPF / DMARC)

Explain the concept: SPF and DMARC are DNS-based mechanisms that let a domain declare who is allowed to send mail on its behalf and what to do with mail that fails. Checking them against the Return-Path domain tells you whether the sending server was authorized.

Return-Path domain identified: [fill in]

SPF Record

Command / tool: dig txt [domain] (or MXToolbox SPF lookup)

[paste the full SPF record here, e.g. v=spf1 include:... -all]

What it means: [briefly interpret — which servers are authorized, and whether the originating IP falls within them].

DMARC Record

Command / tool: dig txt _dmarc.[domain] (or MXToolbox DMARC lookup)

[paste the complete DMARC record here, e.g. v=DMARC1; p=...; rua=...]

What it means: [interpret the policy — p=none/quarantine/reject — and what the domain instructs receivers to do with failing mail].

Analysis: Tie it together. Even a domain with strong SPF/DMARC can be abused via display-name spoofing or lookalike domains. Note what the authentication picture actually tells you about this message.

Phase 4 — Attachment Analysis

Explain: attachments are the most common phishing payload. You extract the file, hash it to create an IOC, then check that hash against threat-intel sources without detonating the file.

Attachment filename: [fill in] Found in the Content-Disposition / Content-Type MIME header.

SHA256 hash:

[paste output of: sha256sum <file>]

VirusTotal findings:

Field	Value
Detection ratio	[e.g. 34/64]
File size	[e.g. 122.31 KB]
Claimed extension	[fill in]
Actual file type	[fill in]

Analysis: The key finding here is often the mismatch between the claimed extension and the actual file type — e.g. a file presented as a document that is really an executable or script. Explain what the real file type is and why that matters. Note the detection ratio, but be honest about what it does and doesn't prove: a low ratio doesn't guarantee safety, and a high one confirms known-malicious.

Indicators of Compromise (IOCs)

A clean IOC table is what makes a writeup useful to other analysts — it's the reusable output of your investigation.

Type	Indicator
Sender address	[fill in]
Reply-To address	[fill in]
Originating IP	[fill in]
Attachment filename	[fill in]
SHA256	[fill in]
[Return-Path domain]	[fill in]
Verdict

State your conclusion clearly: legitimate or phishing, and the evidence that drove it. Reference the strongest indicators — e.g. Reply-To mismatch, IP origin inconsistent with the claimed sender, attachment file-type spoofing. This is the "so what" of the whole investigation.

Conclusion: [Phishing / Legitimate]

Primary evidence:

[strongest indicator]
[second indicator]
[third indicator]
Defensive Takeaways

What would stop or catch this in a real environment? Shows you think beyond the single email to controls and process. Examples to draw from:

User-reported escalation worked — the executive noticed the behavioral mismatch. Reinforces the value of SETA/awareness training.
[Attachment sandboxing, DMARC enforcement at the gateway, display-name spoofing detection, etc.]
What I Learned

One short honest paragraph. What technique was new to you, what you'd do faster next time, or what concept clicked. This personalizes the writeup and reads far better to a hiring manager than a dry answer dump.

Writeup by Chris Janciga · [GitHub profile link] · [date]
