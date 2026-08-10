# TryHackMe – Management Wants a Word

- **Event:** Hacker Holidays 2026 — The Byte Lotus Hotel
- **Category:** Forensics
- **Difficulty:** Hard
- **Points:** 120
- **Completed:** 10 August 2026

> **Status:** Passwords and the flag are redacted while the Hacker Holidays event is active. All analysis was performed in an authorized TryHackMe training environment.

## Summary

A laptop registered to a guest named Vera was recovered after an early checkout. Before the device was wiped, IT collected a full forensic triage using KAPE.

The challenge required correlating artifacts from several different sources rather than finding the answer in a single file. The investigation involved Windows registry hives, an LSA secret, DPAPI master-key decryption, Chrome's saved-password encryption and, finally, a VeraCrypt-encrypted container.

The complete investigation chain was:

1. Examine the KAPE collection and identify the unidentified `Documents/backup` file.
2. Locate Chrome's `History`, `Login Data`, `Web Data`, and `Local State` files.
3. Extract an encrypted saved credential from Chrome's `Login Data` database.
4. Recover Vera's Windows password from an LSA secret in the offline registry hives.
5. Use the password to decrypt Vera's DPAPI master key.
6. Use the master key to unwrap Chrome's protected AES encryption key.
7. Decrypt the saved Chrome password.
8. Use clues supplied by the room to identify `backup` as a VeraCrypt container.
9. Mount the container using the recovered browser credential.
10. Extract an image from a PDF and recover the flag.

## Initial Triage

The KAPE collection contained a Windows directory structure under `C`, including the user profile:

```text
C/Users/vera
```

An initial examination of Vera's `Documents` directory revealed a file named `backup` with no extension.

```bash
file C/Users/vera/Documents/backup
```

```text
C/Users/vera/Documents/backup: data
```

The `file` utility could not identify a known format. Examining the beginning of the file also revealed no recognizable file signature:

```bash
xxd -l 64 C/Users/vera/Documents/backup
```

```text
00000000: f372 f7cc d607 4b17 a8aa 8865 12af abdf  .r....K....e....
00000010: f293 9a74 72ea acbc bee5 b479 4c88 5c7f  ...tr......yL.\
00000020: 5f53 fc44 2988 f8fc ae98 21dd c26a 2a9b  _S.D).....!..j*.
00000030: 8c45 8f73 8ac4 1cdc 8377 bb46 9636 4807  .E.s.....w.F.6H.
```

The apparently high-entropy data was consistent with either encrypted or compressed content. High entropy alone was not enough to identify the file as an encrypted container, but the later room and browser clues supported that conclusion.

## Browser Artifacts

The room's story suggested that the browser had remembered something important. A search for common Chrome artifacts located a non-standard Chrome installation named `Chrome For Testing`:

```bash
find C/Users/vera -type f \
  \( -iname 'Login Data' \
  -o -iname 'Local State' \
  -o -iname 'Web Data' \
  -o -iname 'History' \) -print
```

```text
C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data/Default/History
C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data/Default/Login Data
C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data/Default/Web Data
C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data/Local State
```

These files serve different purposes:

- `History` contains visited URLs, searches, and download history.
- `Web Data` contains autofill information and related browser data.
- `Login Data` contains saved usernames and encrypted passwords.
- `Local State` contains Chrome configuration data, including the protected key used for saved credentials.

### Inspecting Autofill Data

The `Web Data` file is a SQLite database. Its `autofill` table contained only the username:

```sql
SELECT name, value, date_created, date_last_used
FROM autofill
ORDER BY date_last_used DESC;
```

```text
username|VeraSecretVault|1784501610|1784501610
```

No plaintext password was present.

### Inspecting Browser History

The `History` database revealed several Google searches and visits to a local service:

```bash
sqlite3 \
  'C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data/Default/History' \
  "SELECT url, title, last_visit_time FROM urls ORDER BY last_visit_time DESC;"
```

Relevant entries included:

```text
http://bytelotus.thm:8080/       SecureVault Portal
http://bytelotus.thm:8080/login  Error response
```

The browser history confirmed the target service but did not expose the password directly.

### Inspecting Saved Credentials

Chrome stores saved credentials in the `logins` table of the `Login Data` SQLite database:

```bash
sqlite3 \
  'C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data/Default/Login Data'
```

The encrypted credential was extracted using:

```sql
SELECT
    origin_url,
    action_url,
    username_value,
    hex(password_value) AS encrypted_password
FROM logins;
```

The result contained one saved login:

```text
origin_url:        http://bytelotus.thm:8080/
action_url:        http://bytelotus.thm:8080/login
username_value:    VeraSecretVault
encrypted_password: 763130C88A72A64F35F63E883EA0A7F64A6870E46B0BBB469A756EDA88B7E324C3E1C51015AA6FD8D65AC48961E1EA324CE1707807FEB3D7
```

The first three bytes were:

```text
76 31 30
```

These bytes decode to the ASCII string:

```text
v10
```

This identifies the value as a Chrome `v10` credential blob. In this format, the password is encrypted using AES-GCM with a Chrome encryption key stored in `Local State`. On this Windows system, that Chrome key was itself protected using Windows DPAPI.

The dependency chain was therefore:

```text
Windows password
    → DPAPI master key
        → Chrome AES key
            → Saved browser password
```

## Recovering the Windows Password

The room's story contained the following hint:

> “Not every hidden file needs a password cracker, some of them just need a really good memory.”

This suggested that the required password was stored somewhere on the system rather than requiring brute-force cracking.

The KAPE collection contained the offline Windows registry hives:

```text
C/Windows/System32/config/SAM
C/Windows/System32/config/SYSTEM
C/Windows/System32/config/SECURITY
```

These were processed using Impacket's `secretsdump`:

```bash
impacket-secretsdump \
  -sam C/Windows/System32/config/SAM \
  -system C/Windows/System32/config/SYSTEM \
  -security C/Windows/System32/config/SECURITY \
  LOCAL
```

The output included local account hashes, DPAPI system keys and an LSA secret named `DefaultPassword`:

```text
[*] DefaultPassword
(Unknown User):[REDACTED]
```

The recovered `DefaultPassword` entry was stored as an LSA secret in the offline `SECURITY` hive. This is distinct from, although related in purpose to, the classic Winlogon registry value with the same name.

Such secrets may be created to support automatic logon and can expose a stored password to anyone with sufficient access to the relevant registry hives.

```text
Recovered Windows password: [REDACTED]
```

This matched the room's hint: the password did not need to be cracked because Windows had remembered it.

## Identifying the DPAPI Master Key

DPAPI master keys belonging to a Windows user are normally stored inside the user's `Protect` directory.

The following command located Vera's master-key file:

```bash
find C/Users/vera/AppData/Roaming/Microsoft/Protect \
  -type f -printf '%f %p\n'
```

```text
c90719ef-5b98-474e-b934-136d606a702a C/Users/vera/AppData/Roaming/Microsoft/Protect/S-1-5-21-2529683458-431225740-1723070931-1000/c90719ef-5b98-474e-b934-136d606a702a

Preferred C/Users/vera/AppData/Roaming/Microsoft/Protect/S-1-5-21-2529683458-431225740-1723070931-1000/Preferred
```

This revealed both the user SID and the master-key GUID:

```text
SID:  S-1-5-21-2529683458-431225740-1723070931-1000
GUID: c90719ef-5b98-474e-b934-136d606a702a
```

## Decrypting the DPAPI Master Key

With Vera's SID and recovered Windows password, the DPAPI master-key file could be decrypted offline:

```bash
impacket-dpapi masterkey \
  -file 'C/Users/vera/AppData/Roaming/Microsoft/Protect/S-1-5-21-2529683458-431225740-1723070931-1000/c90719ef-5b98-474e-b934-136d606a702a' \
  -sid S-1-5-21-2529683458-431225740-1723070931-1000 \
  -password '[REDACTED]'
```

This returned the decrypted DPAPI master key required to unwrap Chrome's protected encryption key.

The master-key value itself has been omitted because publishing it adds no educational value.

## Decrypting Chrome's Encryption Key

Chrome stores its protected password-encryption key in the `os_crypt` section of `Local State`.

The value can be extracted and Base64-decoded using Python:

```python
import base64
import json

local_state_path = (
    "C/Users/vera/AppData/Local/Google/"
    "Chrome For Testing/User Data/Local State"
)

with open(local_state_path, "r", encoding="utf-8") as file:
    local_state = json.load(file)

protected_key = base64.b64decode(
    local_state["os_crypt"]["encrypted_key"]
)

print(protected_key.hex())
```

The decoded value began with the literal bytes:

```text
44 50 41 50 49
```

These bytes represent the ASCII string:

```text
DPAPI
```

Chrome adds this five-byte prefix to identify the remaining content as a DPAPI-protected blob.

After removing the prefix, the remaining blob was decrypted using the DPAPI master key recovered in the previous step. This produced Chrome's 32-byte AES key.

```text
Recovered Chrome AES key: [REDACTED]
```

<img width="1377" height="859" alt="chrome AES key" src="https://github.com/user-attachments/assets/be7e57c7-40b7-4a3e-a5c8-414103bc67d1" />


## Decrypting the Saved Password

The encrypted password extracted from `Login Data` used Chrome's `v10` AES-GCM structure:

```text
3-byte prefix | 12-byte nonce | ciphertext | 16-byte authentication tag
```

The core decryption logic was:

```python
from Cryptodome.Cipher import AES

encrypted_password = bytes.fromhex(
    "763130C88A72A64F35F63E883EA0A7F64A6870E46B0BBB469A756EDA88B7"
    "E324C3E1C51015AA6FD8D65AC48961E1EA324CE1707807FEB3D7"
)

assert encrypted_password[:3] == b"v10"

nonce = encrypted_password[3:15]
ciphertext = encrypted_password[15:-16]
authentication_tag = encrypted_password[-16:]

# aes_key is the 32-byte Chrome key recovered from Local State.
cipher = AES.new(aes_key, AES.MODE_GCM, nonce=nonce)

plaintext = cipher.decrypt_and_verify(
    ciphertext,
    authentication_tag
)

print(plaintext.decode("utf-8"))
```

The authentication tag verified successfully, confirming that the correct AES key had been recovered.

```text
Recovered browser credential: [REDACTED]
```

<img width="1316" height="786" alt="Successful Chrome credential decryption with the plaintext password" src="https://github.com/user-attachments/assets/c1616cb9-72b7-47f8-9bf0-f18ed82accaa" />


Although the credential was stored against the web login at `http://bytelotus.thm:8080/login`, its real purpose in the challenge became apparent when it was correlated with the unidentified `backup` file.

## Identifying the VeraCrypt Container

The saved username was:

```text
VeraSecretVault
```

The combination of `Vera`, `SecretVault`, and the high-entropy `backup` file suggested that the file could be a VeraCrypt container.

The room itself also provided an intentional clue in its social-media story:

> “also why did Patch tell me this version number 1.26.29 idk what it means :(”

The unusually specific version number `1.26.29` matched a VeraCrypt release. Because this value was deliberately supplied in the room's hint, it provided the strongest confirmation that the otherwise unidentified `backup` file should be treated as a VeraCrypt container.

The version number was therefore an identification clue from the challenge. It was not necessary to claim that only this exact version could open the container.

## Installing VeraCrypt

VeraCrypt was not available in Kali's default package repositories, so version `1.26.29` was downloaded from the official release source:

```bash
cd ~/Downloads

wget \
  https://launchpad.net/veracrypt/trunk/1.26.29/+download/veracrypt-1.26.29-setup.tar.bz2

tar -xvf veracrypt-1.26.29-setup.tar.bz2

sudo ./veracrypt-1.26.29-setup-console-x64
```

The installed version was verified with:

```bash
veracrypt --version
```

```text
VeraCrypt 1.26.29
```

## Mounting the Container

The container was mounted successfully using VeraCrypt:

```bash
mkdir -p /tmp/vera_mount

veracrypt --text \
  --keyfiles= \
  --protect-hidden=no \
  'C/Users/vera/Documents/backup' \
  /tmp/vera_mount
```

> For a stricter forensic workflow, the container should preferably be mounted read-only by adding `--mount-options=ro`. This reduces the risk of modifying the original artefact during examination.

When prompted, the recovered Chrome credential was entered as the container password:

```text
Enter password: [REDACTED]
```
The password was accepted, confirming that the browser credential had been reused as the VeraCrypt passphrase.

### Alternative Mounting Method

A VeraCrypt-compatible container can alternatively be opened using Linux's native `cryptsetup` support:

```bash
sudo cryptsetup tcryptOpen \
  --readonly \
  --veracrypt \
  'C/Users/vera/Documents/backup' \
  vera_backup
```

The unlocked device can then be mounted read-only:

```bash
sudo mkdir -p /mnt/vera
sudo mount -o ro /dev/mapper/vera_backup /mnt/vera
```

After examination, it can be safely unmounted and closed:

```bash
sudo umount /mnt/vera
sudo cryptsetup close vera_backup
```

This was not the method used to obtain the flag during the investigation, but it provides a useful alternative when VeraCrypt itself is unavailable.

## Locating the Flag

The mounted volume contained a directory named:

```text
secret_financial_documents
```

The files were enumerated using:

```bash
find /tmp/vera_mount -type f
```

```text
/tmp/vera_mount/secret_financial_documents/important_invoice_byte_lotus.pdf
/tmp/vera_mount/secret_financial_documents/transactions_q3.csv
```

Direct searches and `pdftotext` did not reveal useful text from the PDF. The document's internal structure was therefore examined:

```bash
pdfimages -list important_invoice_byte_lotus.pdf
```

```text
page num  type   width height
   1    0 image    636   724
   1    1 smask    636   724
```

This showed that the PDF contained an embedded image and a soft-mask layer rather than a searchable text layer.

The image was extracted directly:

```bash
pdfimages -png important_invoice_byte_lotus.pdf invoice_img
```

Visual inspection revealed an invoice from Byte Lotus Resorts. One of the invoice descriptions contained the flag in human-readable form.


<img width="939" height="1020" alt="Extracted Byte Lotus invoice with the flag redacted" src="https://github.com/user-attachments/assets/c27c94d3-5004-4055-9342-761a67871c93" />

```text
Flag: THM{...}
```

> The complete flag is intentionally redacted while the event is active.

No password cracking, steganographic extraction, or OCR was ultimately required for the final step. Once the image had been extracted from the PDF, the flag was visible as part of the invoice itself.

## Technique Summary

This challenge was built around a multi-stage forensic credential-recovery chain:

1. **LSA secrets may contain stored credentials.**  
   The offline `SECURITY` hive contained a recoverable `DefaultPassword` secret associated with automatic logon.

2. **DPAPI protects user-specific secrets.**  
   Vera's Windows password allowed the relevant DPAPI master key to be decrypted. DPAPI-protected data tied to the same credentials may then become decryptable offline when the necessary master keys and supporting artifacts are available.

3. **Chrome adds a second encryption layer.**  
   Chrome protected its own AES key using DPAPI and then used that AES key to encrypt individual saved passwords with AES-GCM.

4. **Password reuse connected separate artifacts.**  
   The browser-saved credential was reused as the VeraCrypt container passphrase.

5. **Room hints supplied technical direction.**  
   The intentionally provided version number `1.26.29` identified VeraCrypt as the tool and format associated with the unidentified `backup` file.

6. **The final PDF contained an image rather than searchable text.**  
   Extracting the embedded image revealed the flag without requiring OCR or further decryption.

## Lessons Learned

- **Not every password needs to be cracked.**  
  The room's hint about a browser having a “really good memory” pointed toward stored credentials and protected artifacts rather than brute-force attacks.

- **Forensic artifacts become more valuable when correlated.**  
  The registry hives, DPAPI master key, Chrome databases, `Local State`, room hints, and encrypted container each provided only part of the answer.

- **High entropy is an indicator, not proof of encryption.**  
  The `backup` file could not be conclusively identified from its bytes alone. Its purpose became clear only after correlating it with the username and the VeraCrypt version hint.

- **Chrome credential recovery requires two decryption layers.**  
  The DPAPI-protected Chrome AES key must be recovered before an individual AES-GCM password blob can be decrypted.

- **Oddly specific challenge details are worth investigating.**  
  The version number `1.26.29` was intentionally included by the room as a clue and directly led to VeraCrypt.

- **Password reuse can expose otherwise protected information.**  
  Reusing a browser credential as an encrypted-volume passphrase allowed access to data that initially appeared unrelated.

- **Forensic examination should preserve the evidence.**  
  Mounting encrypted containers read-only is preferable because it reduces the risk of modifying the original artefact. Although the container was not explicitly mounted read-only during this investigation, `--mount-options=ro` would improve the forensic workflow.

## Tools Used

- KAPE
- `file`
- `xxd`
- SQLite
- Impacket `secretsdump`
- Impacket DPAPI utilities
- Python
- PyCryptodome
- VeraCrypt
- `cryptsetup`
- Poppler `pdfimages`

## Disclaimer

All security testing and forensic analysis documented in this write-up were performed in an authorized TryHackMe training environment. Passwords and the flag have been redacted to avoid spoiling the active challenge.
