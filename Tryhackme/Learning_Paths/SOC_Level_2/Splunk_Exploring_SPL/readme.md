# TryHackMe – Splunk: Exploring SPL

<img width="1305" height="138" alt="Overview" src="https://github.com/user-attachments/assets/4732b547-394a-45b1-9353-bb6ff248b365" />


## Room overview

This room introduces Splunk's Search Processing Language (SPL) and demonstrates how SOC analysts can search, filter, structure, transform and enrich log data. The practical exercises use Windows and VPN logs to explore field-based searches, regular expressions, statistical commands, lookups and basic anomaly detection.


## Learning objectives

- Search and filter events with SPL operators and field-value pairs
- Chain commands with pipes to refine results
- Structure log data using `fields`, `table`, `sort` and `reverse`
- Aggregate events using `top`, `stats`, `chart` and `timechart`
- Enrich events using `iplocation` and lookup tables
- Detect geographic and time-based authentication anomalies

## Task 1 – Introduction

Splunk is a Security Information and Event Management platform used to search, analyse and visualise log data. SPL turns large collections of raw events into useful results for security monitoring and investigations.

No answer was required for this task.

## Task 2 – Search & Reporting

The first search examined all events in the `windowslogs` index:

```spl
index=windowslogs
```

The time range was set to **All time**. The `SourceIP` result was obtained by opening the field in the Fields sidebar and reviewing its most common values.

### Answers

- Total events: `12,256`
- `SourceIP` with the most events: `172.90.12.11`
- Events on 15 April 2022 from 08:05 to 08:06: `134`

## Task 3 – Search Operators

SPL supports relational operators such as `=`, `!=`, `<` and `>`, together with logical operators such as `NOT`, `AND`, `OR` and `IN`. Wildcards allow partial matches, while parentheses control how conditions are evaluated.

Example searches:

```spl
index=windowslogs EventID=4624
```

```spl
index=windowslogs DestinationIp=172.18.39.6 DestinationPort=135
```

```spl
index=windowslogs Hostname=Salena.Adam DestinationIp=172.18.38.5
```

```spl
index=windowslogs cyber*
```

### Answers

- Events with `EventID=4624`: `26`
- Events with destination `172.18.39.6:135`: `4`
- Most frequent `SourceIp` for the specified host and destination: `172.90.12.11`
- Events returned by the term `cyber*`: `12,256`
- Logical operator with the lowest priority: `AND`

Splunk evaluates `OR` before `AND` in search expressions. Parentheses should therefore be used when an explicit evaluation order is required.

## Task 4 – Filtering Results

The `fields` command limits the displayed fields without changing their original values:

```spl
index=windowslogs
| fields Domain SourceProcessId TargetProcessId
```

The maximum process identifier was verified with:

```spl
index=windowslogs
| stats max(SourceProcessId) AS HighestSourceProcessId
```

The following regular expression returned `TargetObject` values ending in `Manager`:

```spl
index=windowslogs
| regex TargetObject="Manager$"
```

### Answers

- Highest `SourceProcessId`: `9496`
- `TargetObject` with the most results: `HKLM\SOFTWARE\Microsoft\SecurityManager`

The `$` character anchors the regular expression to the end of the field value.

## Task 5 – Structuring Results

The `table` command creates a focused view containing only selected fields:

```spl
index=windowslogs
| table EventID AccountName AccountType
```

Adding `reverse` changed the display from newest-first to oldest-first:

```spl
index=windowslogs
| table EventID AccountName AccountType
| reverse
```

A process-creation timeline revealed the creation of a local user account:

```spl
index=windowslogs EventID=1
| table _time ParentProcessId ProcessId ParentCommandLine CommandLine
| reverse
```

Relevant command:

```text
C:\windows\system32\net1 user /add A1berto paw0rd1
```

### Answers

- First `AccountName`: `SYSTEM`
- First `EventID` after applying `reverse`: `800`
- Password assigned to `A1berto`: `paw0rd1`

## Task 6 – Transforming Commands

Transforming commands aggregate raw events into statistical results. The most frequent process image was found with:

```spl
index=windowslogs
| top Image
```

The `iplocation` command enriched source IP addresses with geographic information:

```spl
index=windowslogs
| iplocation SourceIp
| stats count BY Region
```

An external lookup table added risk scores to process images:

```spl
index=windowslogs
| lookup image_riskscore Image OUTPUT RiskScore
| stats count BY Image RiskScore
| sort - RiskScore
```

### Answers

- Most frequently occurring `Image`: `C:\windows\system32\svchost.exe`
- Source IP region: `California`
- `Image` with the highest `RiskScore`: `C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe`

## Task 7 – Anomaly Detection

### Geographic outliers

The first detection calculated how frequently each user logged in from each country. A user-country combination representing less than 10% of that user's logins was treated as anomalous:

```spl
index=vpnlogs
| eventstats count AS logins_by_user BY user
| eventstats count AS logins_by_user_country BY user src_country
| eval country_freq=logins_by_user_country/logins_by_user
| where country_freq < 0.1
| table _time user src_ip src_country country_freq
```

### Time-based outliers

The second detection calculated each user's typical login hour and standard deviation. Events with a z-score greater than three were treated as anomalous:

```spl
index=vpnlogs
| eval hour=tonumber(strftime(_time, "%H")) + tonumber(strftime(_time, "%M"))/60
| eventstats avg(hour) AS typical_hour stdev(hour) AS stdev_hour BY user
| eval zscore=abs(hour-typical_hour)/stdev_hour
| where zscore > 3
| eval hour=round(hour,2), typical_hour=round(typical_hour,2)
| eval stdev_hour=round(stdev_hour,2), zscore=round(zscore,2)
| table _time user src_ip src_country hour typical_hour stdev_hour zscore
| sort - zscore
```

### Answers

- Additional user identified as a geographic outlier: `jsmith`
- Anomalous country for `jsmith`: `JP` – Japan
- User who suspiciously logged in around 03:00: `njackson`

## Key takeaways

- Begin broad searches with an index and time range, then add filters progressively.
- Parsed fields enable precise searches and statistical analysis.
- Tables are useful for timelines and readable incident reports.
- Transforming commands reveal trends that are difficult to identify in raw events.
- Enrichment adds context, but lookup and geolocation results must still be validated.
- Simple frequency calculations and z-scores can expose authentication outliers without requiring a full machine-learning model.

## Conclusion

This room demonstrated how SPL can support the full analytical workflow: locating relevant events, filtering noise, reconstructing timelines, enriching indicators and detecting anomalies. These techniques provide a practical foundation for alert triage, threat hunting and detection engineering in a Splunk-based SOC.

---

> This write-up documents an authorised TryHackMe training room. No real-world systems were tested.
