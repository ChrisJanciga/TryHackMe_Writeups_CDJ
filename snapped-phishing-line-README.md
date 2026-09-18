# TryHackMe — Phishing Kit Analysis Writeup

> **Platform:** TryHackMe
> **Room:** Snapped Phish-ing Line
> **Category:** Phishing Analysis / Email Forensics / CTI / SOC
> **Difficulty:** Easy
> **Date completed:** 9/18/2026
> **Author:** Chris Janciga

---

## Overview

Multiple employees at SwiftSpend Financial reported a suspicious email, and some had already submitted their credentials before the incident was escalated. This writeup documents the investigation: analyzing the phishing emails, following the redirection URLs, retrieving and examining the phishing kit, and using CTI tools to profile the adversary and uncover additional indicators.

**Reported situation:**

- multiple employees across departments received a suspicious email
- some users submitted credentials and lost account access
- potential for wider compromise — escalated for investigation

**Objective:** Determine the scope of the attack and uncover how the adversary operated, documenting supporting evidence throughout.

---

## Tools Used / Can Be Used

| Tool | Purpose |
|------|---------|
| Text editor / `cat` | Reading raw `.eml` headers and email bodies |
| Web browser (in VM) | Opening attachments / phishing pages safely in the lab |
| `sha256sum` | Hashing the phishing kit archive |
| VirusTotal | Reputation check, threat categorization, archive contents |
| CyberChef | Decoding the flag / encoded values |

---

## Investigation

### Phase 1 — Email Triage

> With multiple phishing emails collected, the first step is reviewing them to identify recipients, the sending address, and the initial artifacts that tie the campaign together.

**Method:** Reviewed the emails in the `phish-emails` folder on the desktop.

| Artifact | Detail | Value |
|----------|--------|-------|
| Recipient of "Quote for Services Rendered" email | William McClean | william.mcclean@swiftspend.finance |
| Adversary sending email address | Used to send the phishing emails | Accounts.Payable@groupmarketingonline.icu |

**Analysis:** Note what ties these emails together to point to this as a larger campaign — they all have a similar structure, same sender, same attack method.

---

### Phase 2 — Attachment & Redirection Analysis

> Phishing attachments often don't carry the payload directly — they contain a redirection URL that sends the victim onward to the credential-harvesting page. Extracting that URL from the file exposes the next stage of the attack.

**Attachment investigated:** email addressed to Zoe Duncan

| Artifact | Value |
|----------|-------|
| Root domain of redirection URL | kennaroads.buzz |

**Impersonation:** Opened the attachment in the VM browser (safely, inside the lab).

| Artifact | Value |
|----------|-------|
| Company the login page impersonates | Micorosft |

Copycat domains and redirects are one of the most common ways a phishing attempt will get someone. If someone is in a hurry or it is a part of a common routine they may not look twice at the small indicators of it being illigitimate.

---

### Phase 3 — Exposed Files & Phishing Kit Retrieval

> Attackers frequently leave their toolkit exposed on the same server hosting the phishing page. Browsing to predictable directories (like `/data`) can reveal the phishing kit archive — a goldmine of indicators.

**Directory investigated:** `/data`

| Artifact | Value |
|----------|-------|
| Archive file name | Update365.zip |

**Phishing kit hash:**

Downloaded the archive to the VM and hashed it using `sha256sum`

```
ba3c15267393419eb08c7b2652b8b6b39b406ef300ae8a18fee4d16b19ac9686
```

Note the significance of finding an exposed kit — it's operational security failure by the attacker, and it gives you the actual tooling to analyze. The hash becomes your primary IOC for pivoting.

---

### Phase 4 — Threat Intelligence (VirusTotal)

> Pivoting on the archive's hash in VirusTotal reveals how the security community has categorized it, what's inside it, and links to related infrastructure — without you having to detonate anything.

| Field | Value |
|-------|-------|
| Additional threat category (aside from phishing) | Trojan |
| Number of files contained in the archive | 49 |

**Analysis:** Interpret the extra threat category — what does it tell you about the kit's capabilities beyond credential theft? Note the file count as a measure of the kit's complexity.

---

### Phase 5 — Captured Credentials & Kit Internals

> Exposed phishing kits often log the credentials they capture, and the kit's source code reveals where stolen data is exfiltrated to. Examining the log file and the `submit.php` handler uncovers both victims and the adversary's collection channel.

**Exposed credential log:** `/data/Update365/` log file

| Artifact | Value |
|----------|-------|
| Email of user who submitted credentials more than once | michael.ascot@swiftspend.finance |

**Exfiltration channel:** extracted the archive using `unzip Update365.zip -d phishkit` and located the `submit.php` file

| Artifact | Value |
|----------|-------|
| Adversary's credential-collection email address | m3npat@yandex.com |

The logs reveal who was vulnerable to this attack and who may need better cybersecurity education especially for those who logged into the credential grabber multiple times. The file submit.php revealed the attackers own email which was just one of the OPSec mistakes made by the attacker.

---

### Phase 6 — Flag Retrieval

> The final task is an encoded flag left on the phishing infrastructure, decoded with CyberChef.

**Method:** Retrieved `flag.txt` from the phishing URL by adding `/flag.txt` to the end of office365 and decoded it in CyberChef using from base 64 and reversed

| Artifact | Value |
|----------|-------|
| Decoded secret value | THM{****_****_***_***} |

---

## Indicators of Compromise (IOCs)

| Type | Indicator |
|------|-----------|
| Adversary sending address | Accounts.Payable@groupmarketingonline.icu |
| Adversary collection address | m3npat@yandex.com |
| Redirection root domain | kennaroads.buzz |
| Phishing kit archive name | Update365.zip |
| Phishing kit SHA256 | ba3c15267393419eb08c7b2652b8b6b39b406ef300ae8a18fee4d16b19ac9686 |
| Impersonated brand | Microsoft |
| Compromised user(s) | michael.ascot@swiftspend.finance |
| Compromised user(s) | zoe.duncan@swiftspend.finance |
| Compromised user(s) | derick.marshal@swiftspend.finance |
| Compromised user(s) | michelle.chen@swiftspend.finance |

---

## Conclusion

The attack was a phishing campaign with the goal of harvesting login information from employees in the finance department. The attack orginated from a single point and the attacker left a log exposed exposing their own email address and password.

**Verdict:** Phishing campaign / credential harvesting

---

## Defensive Takeaways

- User reporting caught it — reinforces SETA/awareness training value.
- Credential compromise response: forced password resets for affected users, MFA enforcement, blocking the malicious domain at the proxy/gateway, submitting the kit hash to threat intel feeds, etc.

---

## Techniques Used

Basic email analysis to uncover the phishing attempt investigation lead to a credential harvesting page attemptint to immitate microsoft the attacker left some files exposed in the /data directory that lead to exposed log files and the attackers own email address and password

---

*Writeup by Chris Janciga · 9/18/2026 · formatted by claude opus 4.8
