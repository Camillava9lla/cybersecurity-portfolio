# TryHackMe – Passive Reconnaissance

> Notes and practical walkthrough from the Passive Reconnaissance room in the Jr Penetration Tester learning path.

## Summary

This room introduced passive reconnaissance: gathering intelligence from public sources without directly probing the target's systems. The exercises covered domain registration data with WHOIS and RDAP, DNS queries with `dig` and `nslookup`, passive subdomain discovery through DNSDumpster and Certificate Transparency logs, and internet-facing service intelligence from Shodan.

The practical tasks showed how public records can reveal registration dates, registrars, authoritative name-server providers, DNS configuration, subdomains, hosting infrastructure, and exposed services. These findings are useful to both penetration testers mapping a target's external footprint and defenders identifying unnecessary exposure.

## Room Information

| Field | Details |
| --- | --- |
| Learning path | Jr Penetration Tester |
| Module | Network Reconnaissance |
| Room | Passive Reconnaissance |
| Difficulty | Easy |
| Tools and services | WHOIS, RDAP, `dig`, `nslookup`, DNSDumpster, crt.sh and Shodan |
| Target data | Public TryHackMe-owned domains and third-party indexes |

No target virtual machine was deployed. The exercises used public registration records, DNS data and internet-wide search services.

<img width="586" height="479" alt="Overview" src="https://github.com/user-attachments/assets/aa35b4c9-b808-4e4e-9e45-f0daaee062cf" />


## Passive and Active Reconnaissance

Passive reconnaissance uses existing public information and avoids direct interaction with the target. Examples include reviewing public social-media pages, querying third-party DNS resolvers, searching Certificate Transparency logs and inspecting data already collected by Shodan.

Active reconnaissance interacts directly with the target or its personnel. Examples include pinging a target host, scanning ports, enumerating a web application or using social engineering to obtain information from an employee. These activities may be logged or blocked and require explicit authorisation.

### Task Answers

| Scenario | Answer | Reason |
| --- | --- | --- |
| Visit the company's public Facebook page to find employee names | `P` | Existing public information is being reviewed without contacting the organisation. |
| Ping the company's web server | `A` | An ICMP packet is sent directly to the target infrastructure. |
| Social-engineer the target's IT administrator at a party | `A` | The target's employee is directly engaged to obtain information. |

## WHOIS and RDAP

WHOIS is a registration-data query protocol traditionally accessed over TCP port 43. A domain record may reveal its registrar, registration and expiration dates, status codes, authoritative name servers and abuse contacts. Registrant details are commonly redacted by privacy services.

The legacy command used in the room was:

```bash
whois tryhackme.com
```

The relevant fields showed that `tryhackme.com` was registered on 5 July 2018, used Namecheap as its registrar and used Cloudflare name servers.

### Task Answers

| Question | Answer |
| --- | --- |
| When was TryHackMe.com registered? | `20180705` |
| What is the registrar of TryHackMe.com? | `Namecheap.com` |
| Which company is TryHackMe.com using for name servers? | `Cloudflare.com` |

### RDAP

The Registration Data Access Protocol (RDAP) is the modern replacement for WHOIS for generic top-level domains. It uses HTTPS and returns structured JSON, making the results easier to process consistently in scripts.

```bash
curl -s https://rdap.verisign.com/com/v1/domain/tryhackme.com | jq .
```

Useful fields include `ldhName`, `status`, `entities`, `events` and `nameservers`. Registration data can change, so the date on which a lookup was performed should be recorded when documenting live results.

## DNS Queries with dig and nslookup

DNS records can reveal different parts of a domain's public configuration:

| Record | Purpose |
| --- | --- |
| `A` | Maps a name to an IPv4 address. |
| `AAAA` | Maps a name to an IPv6 address. |
| `CNAME` | Maps an alias to another domain name. |
| `MX` | Identifies mail servers and their priorities. |
| `SOA` | Describes the zone's authority and administrative information. |
| `TXT` | Stores text such as SPF policies and domain-verification values. |

`dig` is generally preferred because its output is easier to inspect and use in scripts, while `nslookup` remains useful for compatibility, particularly on Windows.

```bash
dig @1.1.1.1 thmlabs.com TXT
```
<img width="1451" height="767" alt="txt" src="https://github.com/user-attachments/assets/160808b4-f020-4ca1-bc5e-c5c6c4c1f7b7" />



The command consists of:

| Component | Meaning |
| --- | --- |
| `dig` | DNS query tool |
| `@1.1.1.1` | Cloudflare's public recursive DNS resolver |
| `thmlabs.com` | Domain being queried |
| `TXT` | Requested record type |

Cloudflare is not the owner of the requested record. Its resolver follows the DNS hierarchy to find the authoritative source, or returns a valid cached response. Querying a public resolver avoids sending the original query directly from the tester to the target's authoritative name server, although the resolver may contact that server on the tester's behalf if the answer is not cached.

For a concise response containing only the TXT values:

```bash
dig @1.1.1.1 thmlabs.com TXT +short
```

The same query can be performed with `nslookup`:

```bash
nslookup -type=TXT thmlabs.com 1.1.1.1
```

### Task Answer

```text
THM{a5b83929888ed36acb0272971e438d78}
```

## Passive Subdomain Discovery

A normal DNS query resolves a name that is already known. It does not automatically enumerate unadvertised names such as `blog.example.com` or `dev.example.com`. Passive discovery services address this limitation by collecting information from existing public sources.

### DNSDumpster

DNSDumpster aggregates public DNS and infrastructure data. Its results may include hostnames, resolved IP addresses, name servers, mail servers, TXT records, service banners and a graph of relationships between the discovered assets.

Searching for `tryhackme.com` and reviewing **Services / Banners** showed that the entry with the highest count was:

```text
cloudflare
```

This type of result can identify commonly used infrastructure and technologies, but it does not by itself prove that a service is vulnerable.

### Certificate Transparency Logs

Certificate Transparency logs record publicly trusted TLS certificates. The Subject Alternative Name fields within certificates can reveal the domains and subdomains for which they were issued.

The following crt.sh search uses `%` as a wildcard for subdomains:

```text
https://crt.sh/?q=%25.tryhackme.com
```

CT logs are particularly useful for finding historical or less visible subdomains without sending discovery traffic to those hosts. Results still require validation: a certificate entry may be historical, expired or no longer resolve in DNS.

## Shodan

Shodan continuously scans internet-connected systems and indexes the returned service banners. A tester can search Shodan's existing dataset without scanning the target directly.

Information available in a host result may include:

- IP address and Autonomous System Number
- hosting provider or organisation
- approximate geographic location
- observed ports and services
- product and version strings exposed in banners
- tags derived from the collected data

<img width="1274" height="700" alt="shodan" src="https://github.com/user-attachments/assets/aa85cc56-1fbf-43cd-a08c-dfd1d1726e51" />


Example searches include:

```text
hostname:tryhackme.com
apache
nginx
port:443 country:US
http.component:"wordpress"
```

### Task Answers

The Shodan statistics visible during completion on 9 August 2026 gave the following answers:

| Question | Answer |
| --- | --- |
| First country by number of publicly accessible Apache servers | `United States` |
| Third most common port used for Apache | `8080` |
| Most common port used for nginx | `80` |

Shodan statistics change over time as systems appear, disappear or are rescanned. Recording the observation date distinguishes the completed task answer from a permanent claim about the internet.

## Command Quick Reference

| Purpose | Command |
| --- | --- |
| Query a WHOIS record | `whois tryhackme.com` |
| Query RDAP and format the JSON | `curl -s https://rdap.verisign.com/com/v1/domain/tryhackme.com \| jq .` |
| Query A records | `dig tryhackme.com A` |
| Query MX records through Cloudflare | `dig @1.1.1.1 tryhackme.com MX` |
| Query TXT records through Cloudflare | `dig @1.1.1.1 thmlabs.com TXT` |
| Return only the TXT values | `dig @1.1.1.1 thmlabs.com TXT +short` |
| Query TXT records with nslookup | `nslookup -type=TXT thmlabs.com 1.1.1.1` |
| Perform a reverse DNS lookup | `dig -x 1.1.1.1` |
| Search CT logs for subdomains | Visit `crt.sh` and search for `%.tryhackme.com` |

## Key Takeaways

- Passive reconnaissance can reveal a substantial external footprint without directly probing the target.
- WHOIS and RDAP provide registration and delegation data rather than details about the target's running services.
- A recursive resolver such as `1.1.1.1` retrieves public DNS data; it is not necessarily authoritative for the queried domain.
- `dig` and `nslookup` resolve names already known, while DNS aggregation and CT logs can help discover additional subdomains.
- Shodan exposes observations from previous internet-wide scans, making it valuable for both offensive reconnaissance and defensive exposure management.
- Public records and third-party indexes can be incomplete, historical or temporarily inconsistent, so important findings should be cross-checked.
- Defenders can monitor DNS changes, CT logs and Shodan/Censys results to identify rogue assets, dangling records and unintended exposures.

## Ethical Notice

All activities documented here were performed using public data as part of an authorised TryHackMe training room. Any broader security assessment should remain within its defined legal and contractual scope.
