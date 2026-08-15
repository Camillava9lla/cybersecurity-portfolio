# Senior Security Analyst Intro

<img width="1274" height="145" alt="Forside" src="https://github.com/user-attachments/assets/7934edc1-c054-4ef6-95d0-c9e3bde7504a" />


## Overview

This TryHackMe room introduced the responsibilities, mindset and daily work of a SOC Level 2 analyst. It focused on the transition from alert-focused L1 triage to deeper investigation, detection engineering, incident response, threat intelligence and cross-team coordination.

The practical challenge simulated a day in the life of a senior analyst. It involved investigating a malware infection, tuning a SIEM detection rule and responding to an urgent ransomware threat report.

## Key takeaways

- SOC L2 analysts investigate complex alerts escalated by L1 and take greater ownership of incidents.
- Progression to L2 requires improvement in both technical and soft skills.
- An attacker mindset helps analysts reconstruct incidents and anticipate what may happen next.
- L2 responsibilities may include threat hunting, detection engineering, malware response, incident handling and cooperation with IT.
- Detection rules must be tested and tuned to reduce false positives without hiding genuine threats.
- Threat intelligence must be converted into clear and timely actions for relevant teams.

## Challenge summary

An EDR alert identified beaconing from `LPT-0152`, a laptop used by an external contractor. A SIEM process timeline revealed the following malicious chain:

```text
Microsoft Edge
└── ReleaseNotes.pdf.exe
    └── loader.exe
        └── rundll32.exe beacon.dll,Start
```

`ReleaseNotes.pdf.exe` used a deceptive double extension to appear like a PDF document. The correct response was to isolate the affected laptop, continue the investigation and remove the malware.

The original download had not triggered an alert, revealing a detection gap. A new rule was therefore developed to detect double-extension executables:

```spl
index=windows EventCode=11 TargetFilename IN(*.pdf.exe, *.pdf.lnk, ...)
| table _time host user Image TargetFilename
```

Testing produced hundreds of false positives from a trusted application directory. After consulting IT, the folder was excluded:

```spl
NOT TargetFilename IN(C:\\Program Files\\TryHackMeToolkit\\*)
```

The tuned rule retained the malicious detection while removing the known noise. Matching files were configured to be quarantined.

The final part involved an urgent CTI report about the Akira ransomware group exploiting SonicWall SSL-VPN. The appropriate response was to explain the risk to IT and assist with the recommended temporary mitigation: disabling LDAP within the SSL-VPN feature until a patch became available. The SOC team was also given a concise summary, relevant indicators and instructions to remain vigilant.

## Correct answers

| Question or decision | Correct answer |
|---|---|
| Skills required to become L2 | `both` |
| Does exploring new security areas help you grow? | `Yea` |
| Mindset that helps predict how incidents unfold | `attacker mindset` |
| Response to the confirmed data-stealer infection | Isolate `LPT-0152`, finish the analysis and clean the host |
| Response to the missed double-extension executable | Build a detection rule covering double-extension downloads |
| Response to trusted-folder false positives | Consult IT and apply a folder exclusion to the rule |
| Automated action for a matching suspicious file | Quarantine the file |
| Response to the urgent SonicWall/Akira report | Explain the high risk to IT and help implement the mitigation |
| SOC chat response | Summarize the report, share indicators and ask the team to remain vigilant |
| Challenge flag | `THM{much_more_than_alert_triage}` |

<img width="1533" height="674" alt="Flag" src="https://github.com/user-attachments/assets/3c7824e7-4ecb-4832-b1a8-59b0c1f39190" />


## Conclusion

The room demonstrated that senior SOC work is much more than processing alerts. An L2 analyst must establish root cause, improve detections, coordinate containment and mitigation, communicate risk clearly and take ownership of the organisation's security posture.

---

> All activities described in this write-up were completed in an authorised TryHackMe training environment.
