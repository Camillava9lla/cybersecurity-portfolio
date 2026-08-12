# TryHackMe – Python: Building Scripts

<img width="1269" height="142" alt="Python: Building Scripts" src="https://github.com/user-attachments/assets/2cabc748-a59a-4347-86c9-3b4d18575114" />

> A practical introduction to structuring reliable Python scripts with functions, exception handling, file operations, and libraries.

## Room information

- **Platform:** TryHackMe
- **Room:** Python: Building Scripts
- **Category:** Python / Programming Fundamentals
- **Difficulty:** Easy
- **Estimated time:** 60 minutes

## Overview

This room builds on basic Python concepts by combining them into complete, reusable scripts. The main topics were functions, error handling, file input/output, and importing libraries. These concepts were then applied in a password strength checker that reads a wordlist, evaluates a password, provides feedback, and logs the result without storing the password itself.


<img width="1313" height="469" alt="Overview" src="https://github.com/user-attachments/assets/04706ca0-d56a-4bb9-942a-9988ef756811" />

## Functions

Functions organise logic into reusable blocks. Parameters represent the values a function expects, while arguments are the actual values supplied when the function is called.

```python
def check_length(password, min_length=8):
    return len(password) >= min_length

result = check_length("TryHackMe2025!", 12)
print(result)
```

The `return` keyword sends a result back to the caller. Parameters can also have default values, as shown by `min_length=8`.

Variables created inside a function normally have local scope and cannot be accessed outside that function.

## Error handling

Runtime errors can be handled with `try` and `except`, allowing a program to respond gracefully rather than terminate unexpectedly.

```python
try:
    number = int(input("Enter a number: "))
    print(f"You entered {number}")
except ValueError:
    print("That is not a valid number.")
```

Common exceptions explored in the room included:

| Exception | Cause |
| --- | --- |
| `ValueError` | A value cannot be converted to the requested type |
| `FileNotFoundError` | A requested file does not exist |
| `ZeroDivisionError` | A number is divided by zero |
| `KeyError` | A dictionary key does not exist |
| `IndexError` | A list index is outside the valid range |

Specific exception types should be caught where possible because a generic exception handler may conceal programming errors.

## File input and output

Python uses `open()` with a file mode to read or modify files:

| Mode | Operation |
| --- | --- |
| `r` | Read an existing file |
| `w` | Create or overwrite a file |
| `a` | Append data without removing existing content |

A context manager automatically closes the file when the block ends, including when an error occurs.

```python
common_passwords = []

with open("common_passwords.txt", "r") as file:
    for line in file:
        common_passwords.append(line.strip())
```

The `strip()` method removes the newline character and other surrounding whitespace from each entry. Iterating directly over the file is also more memory-efficient than loading a large file into one string.

## Libraries and imports

The Python standard library provides reusable functionality without requiring additional installation.

```python
import string

password = "S3cure!Pass"

has_uppercase = any(character in string.ascii_uppercase for character in password)
has_digit = any(character in string.digits for character in password)
has_special = any(character in string.punctuation for character in password)
```

Third-party packages are installed with Python's package manager, `pip`, and can then be imported into a script. Security-relevant examples include `requests`, `scapy`, `pwntools`, and `paramiko`.

The room also demonstrated `hashlib` by calculating a SHA-256 hash. Changing even a small part of the input produces a completely different digest.

## Password Strength Checker

The final program combined all concepts from the room. Its workflow was:

1. Load and normalise passwords from `common_passwords.txt`.
2. Ask the user for a password.
3. Reject empty input and allow the user to quit.
4. Evaluate length and character variety.
5. Override the score if the password appears in the common-password list.
6. Display a strength label and improvement suggestions.
7. Append a masked result to `password_log.txt`.

The checker awarded up to five points:

- At least eight characters
- At least twelve characters
- At least one uppercase letter
- At least one digit
- At least one punctuation character

If the password appeared in the common-password list, its score was reset to zero regardless of the other checks.

```python
if password.lower() in common_list:
    score = 0
    feedback = [
        "This password is in the common-passwords list. Choose another."
    ]
```

The password `TryHackMe!2025` met all five scoring conditions and was labelled `Strong`. Because it contains 14 characters, the log contained 14 asterisks:

```text
Password: ************** | Strength: Strong (5/5)
```

Masking the password prevents the plaintext value from being written to the log. In a real application, passwords should not be logged at all, even in masked form if the length itself is unnecessary information.

## Task answers

| Task | Question topic | Answer |
| --- | --- | --- |
| 2 | Return a value from a function | `return` |
| 2 | Default port in `scan(target, port=80)` | `80` |
| 2 | Score for `TryHackMe2025!` | `5` |
| 3 | Result of `int("hello")` | `ValueError` |
| 3 | Opening a missing file | `FileNotFoundError` |
| 3 | Result of `10 / 0` | `ZeroDivisionError` |
| 4 | Context-manager keyword | `with` |
| 4 | Remove a trailing newline | `.strip()` |
| 4 | Passwords loaded from the wordlist | `58` |
| 5 | Python package manager | `pip` |
| 5 | Module containing character constants | `string` |
| 5 | First character of the default SHA-256 hash | `5` |
| 6 | Score for a password in the common list | `0` |
| 6 | Label for `TryHackMe!2025` | `Strong` |
| 6 | Asterisks written for `TryHackMe!2025` | `14` |



## Key takeaways

- Functions make scripts easier to reuse, test, and maintain.
- Targeted exception handling prevents expected errors from crashing a program.
- Context managers provide a safe and clean way to work with files.
- Standard and third-party libraries reduce the need to recreate existing functionality.
- Sensitive values such as plaintext passwords must never be written to logs.
- A simple scoring system is useful for learning, but password length, uniqueness, breached-password checks, and multi-factor authentication provide a more meaningful security assessment than character-composition rules alone.

## Conclusion

This room demonstrated the transition from isolated Python syntax to a complete script. The password checker connected user input, reusable functions, control flow, exception handling, wordlist processing, library imports, and safe logging into one working security-related program.
