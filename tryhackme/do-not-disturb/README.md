# TryHackMe – Do Not Disturb

> A practical walkthrough covering web exploitation, Linux enumeration and privilege escalation.


## Lab information

- Platform: TryHackMe
- Event: Hacker Holidays 2026
- Category: Web application and Linux privilege escalation
- Environment: Authorised TryHackMe lab
- Completed: 5th August 2026

## Skills demonstrated

- Web application enumeration
- Session and cookie analysis
- EJS server-side template injection
- Remote command execution
- Reverse shells
- Linux process enumeration
- Node.js Inspector exploitation
- Linux group privilege analysis
- Raw filesystem access with `debugfs`

## Attack path

1. Obtained access to the staff console through a session weakness.
2. Identified unsafe rendering of user-controlled EJS templates.
3. Used the template injection to obtain command execution.
4. Established a reverse shell as the `poolside` user.
5. Discovered a locally exposed Node.js Inspector service.
6. Used the debugger to execute commands as `pipelinesvc`.
7. Identified that `pipelinesvc` belonged to the privileged `disk` group.
8. Read protected filesystem content directly from the raw block device.

## Flags

Flags are redacted to avoid publishing challenge answers.

- User flag: `THM{REDACTED}`
- Root flag: `THM{REDACTED}`
