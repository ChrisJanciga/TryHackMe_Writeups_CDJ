# TryHackMe — WarZone 1 Writeup

> **Platform:** TryHackMe
> **Room:** [WarZone 1](https://tryhackme.com/room/warzone1)
> **Category:** SOC / Alert Triage / Network Forensics
> **Difficulty:** Medium
> **Date completed:** 9/21/2026
> **Author:** Chris Janciga

---
## Overview

An IDS/IPS alert fired for "Potentially Bad Traffic" and "Malware Command and Control Activity." As a Tier 1 analyst at an MSSP, the task is to triage the PCAP, extract artifacts, and determine whether the alert is a true positive. This writeup documents that triage — pulling the alert signature and IOCs, pivoting through VirusTotal for attribution, and retracing the full attack chain.

**Scenario role:** Tier 1 SOC Analyst (MSSP)
**Objective:** Inspect the PCAP, extract artifacts, and confirm whether the C2 alert is a true positive.

> **Note on scope:** This is a methodology writeup. It documents how each artifact was found and how the alert was validated, rather than serving as a plain answer key. Live IOCs (malicious IPs, domains, hashes) are defanged throughout, following standard analyst practice for handling indicators safely.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Wireshark | Deep packet inspection, following streams, reading user-agents |
| Brim / Zeek | Fast PCAP querying, alert and connection summaries |
| NetworkMiner | Artifact extraction — files, hosts, sessions |
| VirusTotal | Reputation, attribution, and pivoting on IOCs |

---

## Triage

### Phase 1 — Reading the Alert

*The alert itself is the starting thread. Open the PCAP in a tool that surfaces IDS alerts (Brim/Suricata alerts or Wireshark) and read the signature that fired for the C2 activity.*

| Artifact | Value |
|----------|-------|
| C2 alert signature | `ET MALWARE MirrorBlast CnC Activity M3` |

**Method:** Imported `Zone1.pcap` into Brim, then filtered the alert events to isolate the C2 signature:
```
event_type=="alert" | "Malware Command and Control Activity Detected" | cut alert.signature
```
This surfaced the Suricata/Emerging Threats rule that fired, giving both the signature name and the malware it maps to.

---

### Phase 2 — Extracting the Core IOCs

*Every alert has a source and destination. These are the first indicators to pull and defang.*

| Artifact | Value (defanged) |
|----------|-----------------|
| Source IP | `172[.]16[.]1[.]102` |
| Destination IP (C2) | `169[.]239[.]128[.]11` |

**Method:** The alert event in Brim lists both endpoints of the flagged connection. The source (`172.16.1.102`) is an internal RFC 1918 address — the victim host on the monitored network — while the destination (`169.239.128.11`) is the external C2 server the malware was beaconing to. Identifying which side is internal vs. external is what separates the victim from the adversary infrastructure.

> **Defanging reminder:** replacing `.` with `[.]` and `http` with `hxxp` renders an indicator non-clickable so it can't be accidentally visited or auto-fetched. Standard practice when documenting live malicious infrastructure.

---

### Phase 3 — Threat Intelligence Pivot (VirusTotal)

*Pivoting the destination IP through VirusTotal turns a raw indicator into attribution — who's behind it and what malware it's tied to.*

| Field | Value |
|-------|-------|
| Threat group attributed (VT Community) | `TA505` |
| Malware family | `MirrorBlast` |
| Majority file type under "Communicating Files" (domain search) | `Windows Installer` |

**Method:** Searched the C2 destination IP in VirusTotal, then read the **Community** tab for analyst attribution (TA505). The **Relations** data tied the infrastructure to the MirrorBlast malware family. Pivoting from the associated domain into **Communicating Files** showed the majority were Windows Installer (`.msi`) files — consistent with the downloads recovered later in the capture.

**Analysis:** The attribution chain is what turns a noisy alert into a confirmed threat. TA505 is a financially motivated threat group known for large-scale email campaigns, and MirrorBlast is a known TA505 tool that abuses malicious `.msi` installers to gain execution. Seeing that pairing — a known group plus a known malware family, delivering the exact file type (`.msi`) observed in the traffic — strongly supports a true-positive determination before I've even looked at the payloads.

---

### Phase 4 — Host & Traffic Analysis

*The user-agent string in the malicious traffic is both an identifying artifact and often a tell — malware frequently uses distinctive or malformed user-agents.*

| Artifact | Value |
|----------|-------|
| User-agent in flagged traffic | `REBOL View 2.7.8.3.1` |

**Method:** Filtered the flagged host's web traffic in Wireshark (`http.user_agent` on the victim IP) and read the User-Agent header from the HTTP requests.

**Analysis:** `REBOL View` is a dead giveaway. REBOL is an obscure scripting language / runtime that no normal user's browser would ever report as its user-agent — legitimate web traffic identifies as Chrome, Firefox, Edge, etc. MirrorBlast is known to use REBOL scripts for its second stage, so this user-agent is effectively a fingerprint of the malware itself calling out. A non-standard runtime as a User-Agent is exactly the kind of anomaly an analyst learns to flag on sight.

---

### Phase 5 — Retracing the Attack Chain

*A real intrusion is rarely a single connection. Retracing the traffic reveals additional infrastructure and the files pulled down during the attack.*

**Additional infrastructure:**

| Artifact | Value (defanged, numerical order) |
|----------|----------------------------------|
| Other IP #1 | `185[.]10[.]68[.]235` |
| Other IP #2 | `192[.]36[.]27[.]92` |

**Downloaded files** *(in the same order as the IPs above):*

| Source IP | Downloaded file |
|-----------|----------------|
| IP #1 | `filter.msi` |
| IP #2 | `10opd3r_load.msi` |

**Method:** Used NetworkMiner's Files tab (and cross-checked with Brim's file-extraction query / Wireshark Export Objects) to reconstruct the transferred objects from the capture. This exposed two additional external hosts serving `.msi` payloads beyond the initial C2 — the staging/download infrastructure. Both downloads being Windows Installer files lines up directly with the "Communicating Files" majority type found in VirusTotal, confirming the same campaign.

---

### Phase 6 — Payload Behavior (Dropped Files)

*Following each downloaded file's traffic reveals what it writes to disk — the dropped payloads and their install paths, key host-based IOCs for the endpoint team.*

**First downloaded file (`filter.msi`) — dropped artifacts:**

| Artifact | Value |
|----------|-------|
| Directory path | `C:\ProgramData\001` |
| File 1 | `arab.exe` |
| File 2 | `arab.bin` |

**Second downloaded file (`10opd3r_load.msi`) — dropped artifacts:**

| Artifact | Value |
|----------|-------|
| Directory path | `C:\ProgramData\Local\Google` |
| File 1 | `rebol-view-278-3-1.exe` |
| File 2 | `exemple.rb` |

**Method:** Inspected the reconstructed traffic for each `.msi` to recover the files written to disk and their install directories. The second installer's payload is especially telling — it drops `rebol-view-278-3-1.exe` (the REBOL runtime seen in the Phase 4 user-agent) alongside `exemple.rb` (a REBOL script), which is the mechanism behind the malicious `REBOL View` beaconing. The first installer stages its executable and a binary blob under a generic `ProgramData` folder, a common living-off-the-land drop location that blends in with legitimate application data.

---

## Indicators of Compromise (IOCs)

*The reusable output of the triage — everything the SOC/IR team would pivot on or block. Defanged.*

| Type | Indicator |
|------|-----------|
| Victim (source) IP | `172[.]16[.]1[.]102` |
| C2 destination IP | `169[.]239[.]128[.]11` |
| Additional IP #1 | `185[.]10[.]68[.]235` |
| Additional IP #2 | `192[.]36[.]27[.]92` |
| Malware family | MirrorBlast |
| Threat group | TA505 |
| User-agent | `REBOL View 2.7.8.3.1` |
| Downloaded files | `filter.msi`, `10opd3r_load.msi` |
| Dropped files (installer 1) | `C:\ProgramData\001\arab.exe`, `C:\ProgramData\001\arab.bin` |
| Dropped files (installer 2) | `C:\ProgramData\Local\Google\rebol-view-278-3-1.exe`, `C:\ProgramData\Local\Google\exemple.rb` |

---

## Verdict

**Determination:** True Positive

**Supporting evidence:**
1. The C2 alert signature (`ET MALWARE MirrorBlast CnC Activity M3`) maps to a known, named malware family rather than a generic heuristic — a strong initial indicator.
2. VirusTotal attributes the C2 IP to the TA505 threat group and the MirrorBlast malware family, with associated infrastructure serving the same `.msi` file type observed in the capture.
3. The traffic shows the full attack chain playing out: a REBOL-based malware user-agent, two additional download servers delivering malicious `.msi` installers, and those installers dropping executables and scripts to disk — behavior consistent with MirrorBlast's known TTPs.

Taken together, this is not benign or a misfire — the alert reflects a genuine malware C2 infection and is confirmed a true positive.

---

## Analyst Actions / Recommendations

*What a Tier 1 analyst does next after confirming a true positive — shows understanding of the SOC workflow beyond the technical triage:*
- Escalate to Tier 2 / IR with the full IOC list and a timeline of the connections.
- Recommend blocking the C2 IP and both download-server IPs at the perimeter firewall/proxy.
- Flag the victim host (`172.16.1.102`) for isolation and hand the dropped-file paths to the endpoint team for host-based investigation and remediation.
- Submit the IOCs (IPs, filenames, hashes) to internal threat-intel feeds and verify detection coverage for MirrorBlast / TA505 TTPs.
- Recommend a retro-hunt across other monitored clients for the same signature, user-agent, and drop paths, since MSSP environments share threat exposure.

---


This room was a strong end-to-end reps on the SOC triage workflow — starting from a single alert and building it into a confirmed, attributed infection. The most useful takeaway was the VirusTotal pivot chain: how a raw C2 IP becomes threat-group attribution (TA505) and a named malware family (MirrorBlast) by working the Community and Relations data, and how that context confirms a verdict before you've even opened the payloads. The REBOL user-agent was a good lesson in spotting anomalies by intuition — a runtime no browser would ever report is an instant flag. Next time I'd lean on Brim earlier for the file-extraction queries rather than bouncing between three tools, to retrace the download chain faster.

---

*Writeup by Chris Janciga · 9/21/2026 · formatted using claude opus 4.8*
