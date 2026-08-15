# TryHackMe – Defensive Security Trends

<img width="1274" height="143" alt="Defensive Security Trends" src="https://github.com/user-attachments/assets/adebf51d-779a-4c46-8f6d-8d13766fc69a" />


## Room overview

This room explores how the modern threat landscape is changing and what these developments mean for defensive security teams. The main topics include faster and more complex attacks, abuse of valid accounts, supply chain compromises, legitimate remote-management tools, and the growing influence of artificial intelligence.

All activities were completed in an authorised TryHackMe training environment.

## Key takeaways

- Modern intrusions can progress from initial access to ransomware deployment in minutes or hours, making rapid containment and automation increasingly important.
- Attackers frequently blend into normal activity by abusing legitimate tools such as AnyDesk, ScreenConnect, TruffleHog and cloud APIs.
- Valid accounts obtained through infostealers and Initial Access Brokers can bypass several traditional security controls.
- Supply chain attacks compromise trusted software, dependencies or service providers instead of targeting the final victim directly.
- Supply chain intrusions still use familiar tactics, techniques and procedures that can be mapped to MITRE ATT&CK.
- AI can improve enrichment, deobfuscation, automation and reporting, but analysts must remain responsible for final decisions.

## Room answers

### Task 1 – Introduction

- Proceed to the next task: `Completed`

### Task 2 – Attacks Become Faster

- Reported average time-to-ransomware: `17 hours`
- What should SOCs do with the triage routine to speed up response? `Automate it`

The main defensive lesson is to reduce the delay between detection and containment. Repetitive triage activities should be automated where practical, while analysts focus on investigation and response.

### Task 3 – Attacks Become Complex

- Remote-access tool used by DarkGate malware: `AnyDesk`
- Is the network perimeter becoming more predictable? `Nay`

Modern environments no longer have one clearly defined perimeter. Remote workers, cloud platforms, SaaS applications, personal devices and third-party dependencies all create additional paths into an organisation.

### Task 4 – Attacks via Valid Accounts

- Incidents involving valid accounts: `39%`
- Criminals who sell access to organisational networks: `Initial Access Brokers`

Infostealers can collect more than usernames and passwords. Browser sessions, authentication tokens, SSH keys and other material may allow attackers to bypass MFA and return long after the original infection.

### Task 5 – Supply Chain Attacks

- First malware link in the Vercel incident: `Lumma Stealer`
- Would the attack TTPs fundamentally differ from other intrusions? `Nay`

Although the initial access route is different, the later stages of a supply chain attack still use recognisable behaviours. Broad telemetry and detection coverage remain essential.

### Task 6 – AI Impact on Cyber Security

- Should AI become the final decision-maker in a SOC? `Nay`
- Which MITRE technique became obsolete because of AI? `None`

AI is useful as an analyst assistant, but its output must be validated. It does not replace foundational knowledge, SIEM and EDR visibility, security hardening, or human accountability.

### Task 7 – Research Challenge

- Backdoored VS Code extension: `Nx Console`
- Open-source package ecosystem at the root of the compromise: `TanStack`
- Year-over-year growth in threat-actor RMM usage: `240%`
- Access type most commonly sold by Initial Access Brokers: `VPN`

The GitHub incident demonstrated a multi-stage supply chain compromise: a poisoned developer dependency ultimately led to a malicious version of a trusted VS Code extension and the compromise of an employee endpoint. The Verizon DBIR findings similarly show how attackers increasingly rely on legitimate RMM software and purchased access rather than obviously malicious tooling.

## Research sources

- [GitHub – Investigation update: unauthorised access to internal repositories](https://github.blog/security/investigating-unauthorized-access-to-githubs-internal-repositories/)
- [CISA – Supply Chain Compromises Impact Nx Console and GitHub Repositories](https://www.cisa.gov/news-events/alerts/2026/05/28/supply-chain-compromises-impact-nx-console-and-github-repositories)
- [Verizon – 2026 Data Breach Investigations Report](https://www.verizon.com/business/resources/T1ae/reports/2026-dbir-data-breach-investigations-report.pdf)

## Conclusion

The room shows that defensive security is not becoming irrelevant as attacks evolve. The fundamentals remain the same, but SOC teams must apply them faster and across a broader environment. Strong visibility, automated containment, identity protection, detection engineering, threat intelligence and informed human decision-making are all necessary to keep pace with modern adversaries.

---

> This write-up documents an authorised TryHackMe learning room. No real-world systems were tested.
