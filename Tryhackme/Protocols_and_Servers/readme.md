# TryHackMe – Protocols and Servers

> A short summary of the seventh room in TryHackMe's Network Security module.

## Room information

* **Learning path:** Jr Penetration Tester
* **Module:** Network Reconnaissance
* **Difficulty:** Easy
* **Completed:** 11 August 2026

## Summary

This room introduced several common network protocols and demonstrated how text-based services can be accessed manually using Telnet.

The practical exercises covered HTTP, FTP, SMTP, POP3 and IMAP. These protocols transmit information in cleartext by default, meaning that credentials, commands and transferred data may be exposed to anyone able to capture the network traffic.

## HTTP

I connected to the web server on TCP port 80 using Telnet:

```bash
telnet <TARGET_IP> 80
```

I then manually requested the flag file using HTTP/1.1:

```http
GET /flag.thm HTTP/1.1
Host: telnet
```

The server returned the requested file in the HTTP response.


<img width="1636" height="1005" alt="flag retrieved using Telnet" src="https://github.com/user-attachments/assets/5a0cac9b-0eaa-4372-ab29-cec8e895519b" />

**Flag:**

```text
THM{e3eb0a1df437f3f97a64aca5952c8ea0}
```

## FTP

I connected to the FTP server using the provided credentials:

```bash
ftp <TARGET_IP>
```

```text
Username: frank
Password: D2xc9CgD
```

After listing the available files, I downloaded the flag:

```text
ls
get ftp_flag.thm
exit
```

The downloaded file was read from the AttackBox terminal:

```bash
cat ftp_flag.thm
```

<img width="800" height="356" alt="FTP flag downloaded and displayed" src="https://github.com/user-attachments/assets/4480ad7e-7037-4ee6-aec5-121b2303c306" />


**Flag:**

```text
THM{364db6ad0e3ddfe7bf0b1870fb06fbdf}
```

## SMTP

I connected to the SMTP service on its default TCP port:

```bash
telnet <TARGET_IP> 25
```

The flag was displayed directly in the SMTP service banner.

**Flag:**

```text
THM{5b31ddfc0c11d81eba776e983c35e9b5}
```

## POP3

I connected to the POP3 service on TCP port 110 and authenticated using Frank's credentials:

```bash
telnet <TARGET_IP> 110
```

The mailbox information was retrieved using the following commands:

```text
USER frank
PASS D2xc9CgD
STAT
LIST
```

The `STAT` command returned the number of available messages and their total size in bytes.

## IMAP

IMAP keeps email stored on the server and synchronises mailboxes across multiple devices. Its default cleartext TCP port is:

```text
143
```

Encrypted IMAPS normally uses TCP port `993`.


<img width="629" height="720" alt="Correct IMAP port answer" src="https://github.com/user-attachments/assets/254f77d2-a243-4bef-9e08-1b93b3e84b5f" />


## Protocol reference

| Protocol | Default port | Secure alternative          | Secure port |
| -------- | -----------: | --------------------------- | ----------: |
| FTP      |           21 | SFTP / FTPS                 |    22 / 990 |
| HTTP     |           80 | HTTPS                       |         443 |
| IMAP     |          143 | IMAPS                       |         993 |
| POP3     |          110 | POP3S                       |         995 |
| SMTP     |           25 | SMTPS / Submission with TLS |   465 / 587 |
| Telnet   |           23 | SSH                         |          22 |

> All testing was performed in an authorised TryHackMe training environment.
