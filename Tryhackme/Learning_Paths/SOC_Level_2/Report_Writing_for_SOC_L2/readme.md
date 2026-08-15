# Report Writing for SOC L2
<img width="1272" height="144" alt="Tryhackme Report Writing for SOC L2" src="https://github.com/user-attachments/assets/bc3210ab-4388-47a7-b72e-7b31e364756c" />

## Overview

This TryHackMe room focused on professional report writing as a core SOC Level 2 skill. L2 analysts must translate technical findings into clear, credible and actionable communication for executives, MSSP customers, internal teams and external DFIR specialists.

## Key takeaways

- L2 bridges the SOC and the outside world through professional reports.
- Reports must be tailored to their intended audience.
- C-level communication should focus on business impact, facts and clear actions while avoiding unnecessary technical jargon.
- Urgent incidents may require an initial report before the full analysis is complete, followed by ongoing communication and a final report.
- DFIR handovers should be technical, chronological, concise and evidence-driven.
- Voice calls are useful for urgent responses, but important decisions should be documented through email or a ticketing system.
- Generative AI can help structure and polish reports, but the analyst remains responsible for accuracy and critical decisions.

## Report audiences

| Audience | Primary focus |
|---|---|
| C-level leadership | Business impact, risk, facts, actions and current status |
| MSSP customer | Urgency, incident summary, required customer actions and next update |
| DFIR team | Context, detailed timeline, scope, actions performed and raw indicators |

## Correct answers

| Question | Answer |
|---|---|
| Which SOC tier bridges the SOC and the outside world? | `L2` |
| What do L2 analysts write to summarise SOC findings? | `Reports` |
| Complete the analysis after sharing the initial SOC report? | `Yea` |
| Keep the SOC team informed about ongoing communication? | `Yea` |
| Executive-report challenge flag | `THM{executive_summary_approved}` |
| Are L2 handover notes intended for a non-technical audience? | `Nay` |
| Section that lists findings chronologically | `Attack Timeline` |
| DFIR-handover challenge flag | `THM{trysaveme_would_be_proud}` |
| What should be provided to AI for the best reports? | `Context` |
| Fully rely on GenAI for critical decisions? | `Nay` |

## Leadership and customer communication

An effective executive or customer-facing report should:

- clearly identify the affected service or system;
- describe the known business impact without exaggeration;
- use professional and unambiguous language;
- recommend proportionate actions;
- state when the next update will be provided;
- keep the SOC team included in the communication trail.

The initial report begins with containment and provides the recipient with immediate actions. Analysis continues afterwards, with deeper findings incorporated into later updates and the final report.

## DFIR handover

DFIR teams need facts and evidence rather than a simplified executive narrative. Useful handover content includes:

- incident context and available log sources;
- a chronological attack timeline;
- affected hosts, users and other assets;
- containment and remediation already performed;
- IP addresses, domains, hashes, files, commands, persistence and other IOCs;
- clear distinction between confirmed facts and analyst assumptions.

The interactive challenge reinforced the need to remove unnecessary text, clarify ambiguous phrasing, identify the actor behind actions, avoid unsupported claims such as confirmed exfiltration and provide context for additional indicators.

## Responsible AI usage

GenAI can reduce the time required to turn investigation notes into a polished report. Good prompts should include sufficient context about the customer, affected assets, threat intelligence, similar historical incidents and monitoring specifics.

However, AI-generated reports must always be reviewed for:

- exposure of sensitive or customer-owned information;
- excessive filler instead of actionable findings;
- hallucinated conclusions caused by missing context;
- unsafe containment recommendations that could damage systems.

The analyst is the decision-maker; AI is only the writing assistant.

## Conclusion

Good report writing turns SOC findings into action. A technically correct investigation has limited value if customers, leadership or DFIR teams cannot understand what happened, assess the impact and respond appropriately.

---

> All activities documented in this write-up were completed in an authorised TryHackMe training environment.
