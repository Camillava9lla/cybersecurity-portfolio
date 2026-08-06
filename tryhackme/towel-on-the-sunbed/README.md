# TryHackMe – Towel on the Sunbed

> Practical walkthrough of an authorised TryHackMe lab.

## Lab information

* **Event:** Hacker Holidays 2026
* **Category:** Web
* **Difficulty:** Medium
* **Vulnerability:** Race condition
* **Completed:** 6 August 2026

## Objective

The objective was to exploit the daily reward mechanism in the Ponzi wellness application and accumulate enough PONZI tokens to access the Whale Vault.

## Reconnaissance

After creating a guest account, I inspected the application's traffic using Burp Suite. Claiming the daily reward generated the following request:

```http
POST /claim HTTP/1.1
Host: 10.114.138.168:3000
Cookie: connect.sid=[REDACTED]
```
<img width="1148" height="1094" alt="Reconnaissance" src="https://github.com/user-attachments/assets/0ab00647-f3d8-4d3c-a7e1-a7235116a03e" />


A normal request awarded 50 PONZI, after which the application prevented another reward from being claimed for 24 hours. Access to the Whale Vault required a balance of 150 PONZI.

## Exploitation

The reward endpoint was vulnerable to a race condition. The server checked whether the daily reward had already been claimed, but concurrent requests could pass this check before the account's claim status was updated.

I duplicated the request in Burp Repeater, placed the requests in a tab group and submitted them concurrently using:


```text
Send group in parallel (last-byte sync)
```
<img width="1132" height="502" alt="Parallel requests sent using Burp Repeater" src="https://github.com/user-attachments/assets/2fb81a7d-54f8-4308-a901-f4e594b80b52" />

Several requests were processed successfully during the same race window. This increased the account balance to 1,250 PONZI, exceeding the 150 PONZI required to unlock the Whale Vault.

## Result

The Whale Vault became accessible and returned the challenge flag:

<img width="653" height="218" alt="THM-Flag" src="https://github.com/user-attachments/assets/5bd6e26b-0377-46cf-b99a-3a373ebd5b5d" />


```text
THM{REDACTED}
```

## Remediation

The application should process reward claims as an atomic database operation. A transaction, row-level lock or unique constraint should ensure that only one reward can be awarded to an account within each eligible period.

## Disclaimer

All testing documented here was performed in an authorised TryHackMe training environment.
