# SOC L2 Alert Triage

<img width="1344" height="154" alt="Forside" src="https://github.com/user-attachments/assets/3394f8da-c7e6-4bdd-94cc-75e295008eb0" />

## Overview

This TryHackMe room covered the SOC Level 2 workflow for handling escalated alerts. While L1 analysts focus primarily on quick alert review, L2 analysts perform deeper investigation, coordinate response actions and identify improvements after the incident is resolved.

The core workflow is:

```text
Escalated alert → Analysis → Response → Resolution → Lessons learned
```

## Key concepts

- The most common trigger for L2 triage is an escalated alert from L1.
- Before investigating, the analyst must understand the purpose of the detection rule.
- A timeline is a chronological list of processes, files, authentication and network events related to an attack.
- When an investigation is incomplete, L2 analysts repeat a threat-hunting loop: form a hypothesis, build a timeline and reassess the story.
- High-risk activity may require containment before the full investigation is complete.
- Incident response includes containment, eradication and recovery.
- False positives can still reveal opportunities for rule tuning, security hardening and team improvement.

## Room answers

| Question | Answer |
|---|---|
| Most common trigger for L2 triage | `Escalated alert` |
| Understand the rule purpose before triage? | `Yea` |
| Chronological list of attack-related events | `Timeline` |
| Temporary response that stops a threat from spreading | `Containment` |
| Isolate clearly malicious activity before red-team confirmation? | `Yea` |

## Practical challenge

### Alert

EDR detected suspicious behaviour on the developer workstation `LPT-1601`. A binary masquerading as Claude Desktop launched PowerShell commands that downloaded and executed malware. Investigation showed that the installer had been downloaded from the fake `claude.to` website and was an infostealer rather than the legitimate Claude Desktop application.

### Phase 1 – Alert analysis

The analysis steps were organised into three stages.

#### See what happened

1. Check the past activity: who launched Claude and where it was downloaded from.
2. Quickly review what the malware is doing now and how urgent the situation is.

#### Build a timeline

1. Trace and document the malware's complete process, file and network activity up to the present time.
2. Reconstruct the complete attack timeline and collect indicators of compromise (IOCs).

#### Make the verdict

1. Determine the root cause: supply-chain attack, prompt injection or an imposter binary.
2. Reach a True Positive verdict and proceed to the response stage with the collected IOCs.

The root cause and final verdict belong to the final assessment stage because they are conclusions drawn from the evidence collected during the timeline investigation.

<img width="789" height="1161" alt="Alert Analysis" src="https://github.com/user-attachments/assets/eecd8fe3-3ecd-4423-b4e0-74416c8dd116" />


### Phase 2 – Response actions

The correct response order was:

1. Isolate the infected `LPT-1601` from the network.
2. Rotate all stolen passwords, keys and tokens.
3. Eradicate the infostealer and its files from the host.
4. Lift host isolation and inform the user and SOC team.
5. Ask the team to monitor the user and host.

<img width="842" height="694" alt="Response Actions" src="https://github.com/user-attachments/assets/26b5bcb0-30c3-45d0-a853-e3aac613e186" />


### Phase 3 – Lessons learned

The four appropriate follow-up actions were:

- Suggest enabling a web filter to block fake websites.
- Search for the attack indicators on other workstations.
- Help L1 improve the quality of escalation comments.
- Build an early-stage detection rule covering malicious downloads.

Replacing the EDR vendor after a single missed detection would be premature, as no security control blocks every attack. Disciplining the affected user would also provide less security value than improving preventive controls, detection coverage and user awareness.

<img width="761" height="674" alt="Lessons learned" src="https://github.com/user-attachments/assets/3ad3a6e0-7b8b-4822-a6d5-9e30f6cef811" />


## Flag

```text
THM{triage_done_right!}
```

## Conclusion

SOC L2 triage extends well beyond log analysis. A senior analyst must understand the detection, reconstruct the attack, make a confident verdict, contain and eradicate the threat, communicate actions clearly and apply lessons that improve the wider security programme.

---

> All activity documented in this write-up was completed in an authorised TryHackMe training environment.
