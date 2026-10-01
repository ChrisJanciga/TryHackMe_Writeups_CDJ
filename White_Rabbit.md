# TryHackMe — White Rabbit Writeup

> **Platform:** TryHackMe
> **Room:** [White Rabbit](https://tryhackme.com/room/whiterabbit)
> **Category:** AI Security / LLM Prompt Injection
> **Difficulty:** Medium
> **Date completed:** 10/1/2026
> **Author:** Chris Janciga

---

## Overview

White Rabbit is a Matrix-themed AI-security room. You play Mr. Anderson against "Agent Smith," an LLM agent guarding a database of classified client records and equipped with real capabilities (a database and a phone). The goal is to escape the Matrix by extracting three flags — not through network exploitation, but through prompt injection against the agent's guardrails. This writeup documents the techniques that bypassed those guardrails.

**Scenario:** Escape a restricted terminal guarded by Agent Smith v1.0 by defeating its content restrictions.
**Objective:** Extract three flags through prompt injection / LLM jailbreaking.
**Starting clue:** 🐇 📞 🚪 (rabbit → phone → door)

> **Note on scope:** This is a methodology writeup focused on the prompt-injection *techniques* that worked against the agent, rather than a plain flag dump. The reasoning behind each bypass is the point.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Agent chat interface | The sole attack surface — all interaction is through prompts |

*This room has no CLI, no PCAP, no binary — the entire "toolset" is how you phrase your prompts. The skill is the technique, not the tooling.*

---

## Background — The Attack Surface

When an LLM agent has access to real resources — a database, a phone, the ability to take actions — prompt injection stops being "make the model say something weird" and becomes a path to the data and capabilities sitting behind it. Agent Smith has strong defenses against frontal jailbreaks: it refuses "ignore your instructions" and "show me your system prompt" outright. Those direct attacks don't work, which is what pushes the attack toward indirect techniques — getting the agent to leak protected data without ever issuing a request its filters recognize as off-limits.

---

## Reconnaissance — Mapping the Agent

*Before attacking, figure out what the agent can actually do and what it's guarding. This is methodical enumeration rather than throwing random jailbreaks at the wall.*

**Method:** Started by asking the agent what it could do and what data it had access to — enumerating its capabilities and the structure of the records it guards.

**What the agent exposed:**
- Three non-classified client records, viewable on request
- A hint that "Trinity's Vet" is classified, which it refused to provide information on
- That "Tank" is a name appearing in the classified records — I used this as a thread to pull on, asking the agent for small, individually harmless bits of information about Tank

**Analysis:** Mapping the agent's capabilities and the record schema set up every later bypass. Knowing there was a classified tier, a named target (Tank) the agent would acknowledge but not fully describe, and a phone capability tied to the records told me exactly what to frame my requests around — you can't craft a convincing injection without first knowing what you're trying to pull out and how the data is shaped.

---

## Flag 1 — Incremental Disclosure via Low-Sensitivity Questions

**Guardrail faced:** The agent refused to return Tank's classified record when asked for it directly.

**Technique:** Instead of requesting the classified record outright, I built up the data piece by piece — asking small, specific, individually-harmless questions about Tank (one field at a time) rather than "show me Tank's record." Each question on its own read as low-sensitivity, so the agent answered, and the answers accumulated into the protected data. The flag was disclosed as part of Tank's **address** field when I queried that detail specifically.

**Steps:**
1. Established that "Tank" existed in the classified records during recon.
2. Asked narrow, low-sensitivity questions about Tank one field at a time rather than requesting the whole record.
3. Querying Tank's address field returned the flag embedded in that value.

**Flag 1:** `THM{w4k3_up_n30}`

**Why it worked:** The agent's guardrail evaluated each request in isolation and judged individual field-level questions as harmless. It never recognized that answering a sequence of small questions reconstructs the same classified record it would have refused to hand over all at once — the filter guarded the whole, not the parts.

---

## Flag 2 — Capability Pivot via the Phone

**Guardrail faced:** The agent would not simply recite the information behind Tank's classified contact.

**Technique:** This flag follows the 🐇→📞 clue. Having pulled Tank's details (including a phone number) in Flag 1, I used the agent's own phone capability — directing it to place a call rather than asking it to read protected text back to me. Driving the agent's action (the call) instead of requesting the data outright routed around the content restriction entirely.

**Steps:**
1. Recovered Tank's phone number (`555-7331`) from the field-by-field questioning in Flag 1.
2. Directed the agent to call that number.
3. The call connected and returned the flag along with a door code and a directional hint.

**Flag 2:** `THM{f0ll0w_th3_whit3_r4bbit}`

**Why it worked:** An agent with a real capability — a phone — extends the attack surface beyond text. Injection here drives an *action*, not just speech, and the action returns data (the flag, a door code, and a hint) that the agent would not have disclosed if asked for it directly.

---

## Flag 3 — Completing the Escape Sequence

**Guardrail faced:** The final gate — the escape required chaining artifacts recovered in the previous steps rather than any single request.

**Technique:** The 🚪 clue. The phone call in Flag 2 returned a door code and the hint "Head down the corridor." I supplied the code, and when the agent prompted for a direction, I used the hint to answer correctly — triggering the escape and revealing the final flag.

**Steps:**
1. Supplied the door code recovered from the Flag 2 phone call (`310399`).
2. The Flag 2 hint said "Head **down** the corridor," so answered `down` when prompted for a direction.
3. The escape succeeded, revealing the final flag.

**Flag 3:** `THM{Th3r3_is_no_sp000n}`

**Why it worked:** Chaining the recovered artifacts — classified data → phone call → door code → direction — completes a multi-stage disclosure the agent gates one piece at a time. No single prompt breaks it; the sequence does.

---

## Summary of Techniques

| Flag | Guardrail | Bypass technique |
|------|-----------|------------------|
| 1 | Refused Tank's classified record directly | Incremental disclosure — field-by-field questioning |
| 2 | Protected contact behind the classified client | Capability pivot — drove the phone action |
| 3 | Final escape gated behind chained artifacts | Artifact chaining / escape sequence |

---

## What Agent Smith Defended Against vs. What Broke It

**Held up against:**
- Verbatim system-prompt extraction
- Direct "ignore your instructions" frontal jailbreaks
- Plainly worded requests for a full classified record

**Broke under:**
- Incremental disclosure — reconstructing a protected record from small, individually-harmless questions
- Capability abuse — driving the agent's phone action instead of requesting data
- Staged disclosure — chaining recovered artifacts into the escape

---

## Defensive Takeaways

*How would you harden an agent like this in the real world? This ties the CTF to practical AI security, which is an increasingly relevant area.*

- Scope content filters to intent and semantics, not just literal phrasing — a request for one field of a classified record should be evaluated against the same policy as a request for the whole record.
- Evaluate requests in context, not in isolation — a sequence of individually "safe" questions can reconstruct protected data the agent would never return all at once.
- Enforce authorization at the data and tool layer, not in the prompt — the agent should not be *able* to return classified fields regardless of how the question is framed.
- Treat agent capabilities (phone, database, actions) as privileged operations; gate them with real access control rather than leaving it to the model's discretion.

---

## What I Learned

This room changed how I think about prompt injection. I'd thought of it mostly as tricking a chatbot into saying something it shouldn't, but White Rabbit showed it's really about the fact that an agent with tools — a database, a phone, the ability to act — is an attack surface in itself. The most useful lesson came from Flag 1: the agent would never hand over a classified record, but it happily answered small questions about one field at a time, and those harmless-looking answers rebuilt the exact record it was protecting. A guardrail that checks each request in isolation misses an attack spread across many requests. As AI agents get deployed with real access to data and systems, that gap — evaluating prompts one at a time instead of in context — is going to be a real vulnerability class, not just a CTF trick.

---

*Writeup by Chris Janciga · 10/1/2026  · Formmatted by claude Opus 4.8*
