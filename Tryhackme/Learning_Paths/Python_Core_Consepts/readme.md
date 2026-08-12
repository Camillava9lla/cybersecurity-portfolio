# TryHackMe – Python: Core Concepts

<img width="1611" height="189" alt="Forside" src="https://github.com/user-attachments/assets/45993be5-bd74-41ec-b735-6e3f8c762b10" />


> A practical introduction to Python data types, collections, operators and loops, with examples relevant to cybersecurity.

## Room information

| Field      | Details                     |
| ---------- | --------------------------- |
| Platform   | TryHackMe                   |
| Room       | Python: Core Concepts       |
| Category   | Python / Security Scripting |
| Difficulty | Easy                        |
| Status     | Completed                   |

## Room overview

| Task | Topic                                   | Main concepts                                                                                     | Status    |
| ---- | --------------------------------------- | ------------------------------------------------------------------------------------------------- | --------- |
| 1    | Introduction                            | Room overview and learning objectives                                                             | Completed |
| 2    | Hello World, Variables and Conditionals | `print()`, variables, data types, conditions, type conversion, f-strings and augmented assignment | Completed |
| 3    | Working with Strings                    | String length, indexing, slicing, methods and character checks                                    | Completed |
| 4    | Lists and Dictionaries                  | Collections, indexing, list methods and key-value pairs                                           | Completed |
| 5    | Arithmetic and Membership Operators     | Exponentiation, floor division, modulus, `in` and `not in`                                        | Completed |
| 6    | Loops: `for` and `while`                | Iteration, `range()`, dictionary loops, `break` and `continue`                                    | Completed |
| 7    | Conclusion                              | Review of the concepts covered in the room                                                        | Completed |

<img width="1614" height="569" alt="Overview" src="https://github.com/user-attachments/assets/8df1c2af-6506-48d2-bc50-685f0e018274" />


## Objective

The objective of this room was to strengthen my understanding of Python fundamentals and learn how core programming concepts can be applied when building security scripts.

The room builds on the earlier *Python: Simple Demo* room. It introduces data types, type conversion, formatted strings, string manipulation, lists, dictionaries, arithmetic operators, membership checks and loops.

These concepts are particularly useful in cybersecurity. A Python script can, for example:

* Process username and password lists
* Inspect passwords against security requirements
* Organise network scan results
* Associate ports with services
* Process IP addresses and log entries
* Automate repetitive analysis

## Task 1 – Introduction

The introductory task presented the purpose and structure of the room.

It explains that the room builds on the basic guessing game from *Python: Simple Demo* and expands into the core concepts needed to create more practical scripts.

The room focuses on understanding how Python stores and processes data, how programs make decisions, and how repeated operations can be automated.

## Task 2 – Hello World, Variables and Conditionals

Task 2 reviewed several concepts introduced in the previous room before adding type conversion, f-strings and augmented assignment.

### Hello World

The simplest Python program uses the `print()` function to display text:

```python
# This is a comment
print("Hello World")
```

The example demonstrates three basic principles:

* Lines beginning with `#` are comments and are ignored during execution.
* The `print()` function displays information in the terminal.
* Text values, known as strings, must be enclosed in quotation marks.

Running `hello.py` on the attached VM produced:

```text
Hello World
```

### Variables and data types

A variable stores a value that can be accessed or changed later in a program:

```python
username = "admin"
port = 443
is_open = True
```

Python determines the data type from the assigned value.

Common data types include:

| Type    | Purpose                      | Example         |
| ------- | ---------------------------- | --------------- |
| `str`   | Text                         | `"admin"`       |
| `int`   | Whole numbers                | `443`           |
| `float` | Decimal numbers              | `3.14`          |
| `bool`  | Logical true or false values | `True`          |
| `list`  | An ordered collection        | `[22, 80, 443]` |

A variable can be updated after it has been created:

```python
age = 30
age = age + 1

print(age)
```

Output:

```text
31
```

The built-in `type()` function reveals the data type of a value:

```python
username = "admin"
port = 8080

print(type(username))
print(type(port))
```

Output:

```text
<class 'str'>
<class 'int'>
```

### Conditional statements

Conditional statements allow a program to make decisions:

```python
port = 443

if port == 443:
    print("HTTPS detected")
elif port == 80:
    print("HTTP detected")
else:
    print("Different service")
```

The main components are:

* `if` checks the first condition.
* `elif` checks an additional condition.
* `else` runs when none of the previous conditions are true.
* A colon marks the beginning of an indented code block.

Common comparison operators include:

| Operator | Meaning                  | Example        |
| -------- | ------------------------ | -------------- |
| `==`     | Equal to                 | `port == 443`  |
| `!=`     | Not equal to             | `port != 80`   |
| `<`      | Less than                | `attempts < 3` |
| `>`      | Greater than             | `score > 10`   |
| `<=`     | Less than or equal to    | `risk <= 5`    |
| `>=`     | Greater than or equal to | `length >= 8`  |

Conditions can be combined using `and`, `or` and `not`:

```python
username = "admin"
is_active = True

if username == "admin" and is_active:
    print("Active administrator account")
```

An important distinction is that `=` assigns a value, while `==` compares two values.

### Type conversion

The `input()` function always returns a string, even when the user enters a number.

The value must therefore be converted before it can be used in a numerical calculation:

```python
text = input("Enter a port number: ")
port = int(text)

print(port + 1)
```

If the user enters `443`, the program prints:

```text
444
```

Useful conversion functions include:

| Function  | Converts to | Example         |
| --------- | ----------- | --------------- |
| `int()`   | Integer     | `int("42")`     |
| `float()` | Float       | `float("3.14")` |
| `str()`   | String      | `str(42)`       |
| `bool()`  | Boolean     | `bool(0)`       |

Without the `int()` conversion, Python would be unable to add the number `1` to the string entered by the user.

### Formatted strings

F-strings provide a readable way to insert variables into text:

```python
username = "admin"
port = 443

print(f"User {username} is on port {port}")
```

Output:

```text
User admin is on port 443
```

This is cleaner than separating the text and variables with commas:

```python
print("User", username, "is on port", port)
```

Running `fstrings_demo.py` demonstrated both f-strings and augmented assignment operators.

The final line printed by the script was:

```text
Scan complete: 192.168.1.1 has 3 open ports
```

### Augmented assignment

Augmented assignment operators provide shorter ways to update existing values:

```python
count = 0

count += 5   # count = count + 5
count -= 2   # count = count - 2
count *= 4   # count = count * 4
count //= 3  # count = count // 3
```

These operators are particularly useful for counters and values that change inside loops.

### Task answers

| Question                                                                 | Answer                                        |
| ------------------------------------------------------------------------ | --------------------------------------------- |
| What built-in function reveals the data type of a value?                 | `type()`                                      |
| If a user enters `3.14` using `input()`, what is its original data type? | `str`                                         |
| What was the final line printed by `fstrings_demo.py`?                   | `Scan complete: 192.168.1.1 has 3 open ports` |

<img width="1936" height="1099" alt="Task 2" src="https://github.com/user-attachments/assets/4cf50e94-43dc-42da-a85d-50ba8fe7e3f0" />


## Task 3 – Working with Strings

A string is a sequence of characters enclosed in quotation marks.

Python supports single, double and triple quotation marks:

```python
single = 'hello'
double = "hello"
multiline = """This string
uses multiple lines."""
```

Strings are important in security scripts because much of the information being processed, including usernames, passwords, IP addresses, URLs and log entries, is represented as text.

### String length

The `len()` function returns the number of characters in a string:

```python
password = "Tr0ub4dor"
length = len(password)

print(f"Password length: {length}")
```

Output:

```text
Password length: 9
```

This can be used to check whether a password or another input meets a minimum length requirement.

### Indexing and slicing

Python indexes begin at `0`.

A single character can be selected using its index:

```python
word = "Python"

print(word[0])
print(word[5])
print(word[-1])
```

Output:

```text
P
n
n
```

A negative index counts backward from the end of the string.

Slicing extracts part of a string using `[start:end]`. The start index is included, while the end index is excluded:

```python
word = "Python"

print(word[0:3])
print(word[2:])
print(word[:4])
```

Output:

```text
Pyt
thon
Pyth
```

Another example is:

```python
word = "TryHackMe"

print(word[3:7])
```

Output:

```text
Hack
```

### Useful string methods

Python provides many built-in methods for processing text:

| Method           | Purpose                          | Example result                      |
| ---------------- | -------------------------------- | ----------------------------------- |
| `.upper()`       | Convert text to uppercase        | `"hello".upper()` → `"HELLO"`       |
| `.lower()`       | Convert text to lowercase        | `"ADMIN".lower()` → `"admin"`       |
| `.strip()`       | Remove whitespace from both ends | `" hi ".strip()` → `"hi"`           |
| `.replace(a, b)` | Replace occurrences of one value | `"cat".replace("c", "b")` → `"bat"` |
| `.split(sep)`    | Split a string into a list       | `"a,b,c".split(",")`                |
| `.startswith(x)` | Check how a string begins        | `"http://".startswith("http")`      |
| `.endswith(x)`   | Check how a string ends          | `"file.txt".endswith(".txt")`       |
| `.count(x)`      | Count occurrences of a value     | `"banana".count("a")` → `3`         |

### Character checks

Character-checking methods return Boolean values and can be used to validate input:

```python
char = "A"

print(char.isupper())
print(char.islower())
print(char.isdigit())
print(char.isalpha())
print(char.isalnum())
```

Output:

```text
True
False
False
True
True
```

The methods have the following purposes:

| Method       | Check                                      |
| ------------ | ------------------------------------------ |
| `.isupper()` | Whether the character is uppercase         |
| `.islower()` | Whether the character is lowercase         |
| `.isdigit()` | Whether the character is a digit           |
| `.isalpha()` | Whether the character is a letter          |
| `.isalnum()` | Whether the character is a letter or digit |

These checks can be used when verifying whether a password contains uppercase letters, lowercase letters or numbers.

### The `in` operator with strings

The `in` operator checks whether a value occurs within a string:

```python
url = "https://tryhackme.com/room/pythoncoreconcepts"

print("tryhackme" in url)
print("hackthebox" in url)
print("https" in url)
```

Output:

```text
True
False
True
```

This can be used when inspecting URLs, file paths, log messages and other textual data.

### Task answers

| Question                                                    | Answer     |
| ----------------------------------------------------------- | ---------- |
| What function returns the number of characters in a string? | `len()`    |
| Given `word = "TryHackMe"`, what does `word[3:7]` return?   | `Hack`     |
| What string method converts `"ADMIN"` to `"admin"`?         | `.lower()` |

<img width="716" height="513" alt="Task 3" src="https://github.com/user-attachments/assets/7e1067a0-8be7-44d2-8cb8-068c6f7cbe63" />


## Task 4 – Lists and Dictionaries

Lists and dictionaries are data structures used to store collections of information.

In cybersecurity, they can be used to store IP addresses, ports, usernames, passwords, services and scan results.

### Lists

A list is an ordered and changeable collection enclosed in square brackets:

```python
ports = [22, 80, 443, 8080]
usernames = ["admin", "root", "guest"]
mixed = ["server1", 443, True]
```

Lists use zero-based indexing and support slicing:

```python
ports = [22, 80, 443, 8080]

print(ports[0])
print(ports[-1])
print(ports[1:3])
```

Output:

```text
22
8080
[80, 443]
```

An item can be changed by assigning a new value to its index:

```python
ports[0] = 2222

print(ports)
```

Output:

```text
[2222, 80, 443, 8080]
```

Common list methods include:

| Method       | Purpose                                |
| ------------ | -------------------------------------- |
| `.append(x)` | Add an item to the end                 |
| `.remove(x)` | Remove the first matching value        |
| `.pop(i)`    | Remove and return the item at an index |
| `.sort()`    | Sort the list in ascending order       |
| `.reverse()` | Reverse the order                      |
| `len(list)`  | Return the number of items             |

Example:

```python
ports.append(3306)
ports.remove(80)
ports.pop(0)
ports.sort()
ports.reverse()
```

The `in` operator also works with lists:

```python
common_passwords = [
    "123456",
    "password",
    "admin",
    "letmein"
]

if "password" in common_passwords:
    print("This password is in the common list.")
```

### Dictionaries

A dictionary stores information as key-value pairs:

```python
services = {
    22: "SSH",
    80: "HTTP",
    443: "HTTPS",
    3306: "MySQL"
}
```

A value can be retrieved through its key:

```python
print(services[22])
print(services[443])
```

Output:

```text
SSH
HTTPS
```

Entries can be added, updated or removed:

```python
# Add a new entry
services[8080] = "HTTP-Alt"

# Update an existing entry
services[22] = "OpenSSH"

# Remove an entry
del services[3306]
```

The resulting dictionary is:

```python
{
    22: "OpenSSH",
    80: "HTTP",
    443: "HTTPS",
    8080: "HTTP-Alt"
}
```

The `in` operator checks whether a key exists:

```python
if 22 in services:
    print(f"Port 22 runs {services[22]}")
```

Useful dictionary methods include:

| Method               | Purpose                               |
| -------------------- | ------------------------------------- |
| `.keys()`            | Return all keys                       |
| `.values()`          | Return all values                     |
| `.items()`           | Return all key-value pairs            |
| `.get(key, default)` | Retrieve a value with a safe fallback |

The `.get()` method is useful when a key may not exist:

```python
service = services.get(9999, "Unknown")

print(service)
```

Output:

```text
Unknown
```

This avoids the error that direct access using `services[9999]` would produce.

### Task answers

| Question                                                                     | Answer      |
| ---------------------------------------------------------------------------- | ----------- |
| What method adds an element to the end of a list?                            | `.append()` |
| Given `services = {22: "SSH", 80: "HTTP"}`, what does `services[80]` return? | `HTTP`      |
| What dictionary method retrieves a value with a safe fallback?               | `.get()`    |

<img width="733" height="522" alt="Task 4" src="https://github.com/user-attachments/assets/8d695017-8e07-400c-9822-8b1bf0613364" />


## Task 5 – Arithmetic and Membership Operators

This task introduced additional arithmetic operators and explored how membership checks can be used with collections.

### Arithmetic operators

In addition to addition, subtraction, multiplication and regular division, Python provides exponentiation, floor division and modulus:

| Operator | Meaning                  | Example   | Result |
| -------- | ------------------------ | --------- | ------ |
| `**`     | Exponentiation           | `2 ** 10` | `1024` |
| `//`     | Floor division           | `10 // 3` | `3`    |
| `%`      | Remainder after division | `10 % 3`  | `1`    |

The modulus operator is useful for determining whether a number is even or odd:

```python
number = 10

if number % 2 == 0:
    print("Even number")
else:
    print("Odd number")
```

Floor division divides and rounds down to the nearest whole number:

```python
print(7 / 2)
print(7 // 2)
```

Output:

```text
3.5
3
```

Exponentiation raises one number to the power of another:

```python
possible_values = 2 ** 8

print(possible_values)
```

Output:

```text
256
```

This is relevant when discussing security concepts such as cryptographic key sizes.

### Membership operators

The `in` and `not in` operators can check membership in strings, lists and dictionaries:

```python
common_passwords = [
    "123456",
    "password",
    "qwerty",
    "letmein"
]

user_password = "qwerty"

if user_password in common_passwords:
    print("This password is too common.")
```

The check can be reversed using `not in`:

```python
if user_password not in common_passwords:
    print("This password is not in the common list.")
```

Membership checks can also be combined with logical operators:

```python
password = "Tr0ubador"
length = len(password)
has_digit = any(char.isdigit() for char in password)

if length >= 8 and has_digit:
    print("Moderate strength")
elif length >= 8 or has_digit:
    print("Weak, but has some merit")
else:
    print("Very weak")
```

The `and` operator requires both conditions to be true, while `or` requires only one condition to be true.

### Task answers

| Question                                           | Answer |
| -------------------------------------------------- | ------ |
| What operator returns the remainder of a division? | `%`    |
| What does `10 // 3` evaluate to?                   | `3`    |
| What does `2 ** 10` evaluate to?                   | `1024` |

<img width="736" height="512" alt="Task 5" src="https://github.com/user-attachments/assets/5be22c0a-2561-4022-a109-2bbfca2b3b09" />


## Task 6 – Loops: `for` and `while`

This task introduced the two main loop types in Python.

Loops make it possible to repeat an operation without writing the same code multiple times.

### While loops

A `while` loop continues running for as long as its condition evaluates to `True`:

```python
attempts = 0
max_attempts = 3

while attempts < max_attempts:
    password = input("Enter password: ")
    attempts += 1
    print(f"Attempt {attempts} of {max_attempts}")
```

The condition is checked before every iteration.

When `attempts` reaches `3`, the condition becomes false and the loop ends.

A `while` loop is useful when the number of required iterations depends on a condition that changes while the program is running.

### For loops

A `for` loop iterates over each item in a sequence.

In a security script, this could be a collection of IP addresses, ports, usernames or log entries:

```python
targets = [
    "192.168.1.1",
    "192.168.1.2",
    "192.168.1.3"
]

for ip in targets:
    print(f"Scanning {ip}...")
```

Output:

```text
Scanning 192.168.1.1...
Scanning 192.168.1.2...
Scanning 192.168.1.3...
```

During each iteration, the variable `ip` temporarily contains the current value from the `targets` list.

### Iterating over strings

Because a string is a sequence of characters, a `for` loop can inspect it one character at a time:

```python
password = "S3cure!"

for char in password:
    if char.isdigit():
        print(f"Found digit: {char}")
    elif char.isupper():
        print(f"Found uppercase: {char}")
```

Output:

```text
Found uppercase: S
Found digit: 3
```

This technique can be used when checking whether a password meets specific requirements.

### The `range()` function

The `range()` function generates a sequence of numbers:

```python
for number in range(5):
    print(number)
```

Output:

```text
0
1
2
3
4
```

As with slicing, the stopping value is excluded. Therefore, `range(5)` produces the numbers from `0` through `4`.

A starting value and step can also be provided:

```python
for number in range(0, 21, 2):
    print(number)
```

This prints every even number from `0` through `20`.

The main forms of `range()` are:

| Syntax                     | Generated sequence                          |
| -------------------------- | ------------------------------------------- |
| `range(stop)`              | From `0` to `stop - 1`                      |
| `range(start, stop)`       | From `start` to `stop - 1`                  |
| `range(start, stop, step)` | From `start`, using the specified increment |

### Iterating over dictionaries

The `.items()` method makes it possible to iterate over dictionary keys and values simultaneously:

```python
services = {
    22: "SSH",
    80: "HTTP",
    443: "HTTPS"
}

for port, service in services.items():
    print(f"Port {port} = {service}")
```

Output:

```text
Port 22 = SSH
Port 80 = HTTP
Port 443 = HTTPS
```

This could be used to process scan results or display a mapping between ports and services.

### Controlling loops with `break`

The `break` keyword immediately terminates a loop:

```python
for port in [22, 80, 443, 8080]:
    if port == 443:
        print(f"Port {port} found. Stopping scan.")
        break

    print(f"Checked port {port}")
```

Output:

```text
Checked port 22
Checked port 80
Port 443 found. Stopping scan.
```

Port `8080` is never checked because the loop ends when port `443` is found.

### Skipping iterations with `continue`

The `continue` keyword skips the remainder of the current iteration and moves to the next item:

```python
lines = ["admin", "", "root", "", "guest"]

for line in lines:
    if line == "":
        continue

    print(f"Processing: {line}")
```

Output:

```text
Processing: admin
Processing: root
Processing: guest
```

This is useful when processing data that may contain blank or irrelevant entries.

### Choosing the appropriate loop

A `for` loop is normally the best choice when iterating over a known sequence, such as a list, string, dictionary or numerical range.

A `while` loop is more suitable when repetition should continue until a runtime condition changes, such as:

* Retrying a login
* Waiting for valid input
* Repeating an operation until a connection succeeds
* Limiting the number of attempts

The practical exercise in `loops_demo.py` demonstrated:

* `for` loops
* `while` loops
* Numerical sequences with `range()`
* Dictionary iteration
* Early termination with `break`
* Skipping iterations with `continue`

The task also included creating a loop that prints every even number from `0` to `20`:

```python
for number in range(0, 21, 2):
    print(number)
```

### Task answers

| Question                                                                 | Answer    |
| ------------------------------------------------------------------------ | --------- |
| What type of loop is best suited for iterating over each item in a list? | `for`     |
| What does `range(3)` produce?                                            | `0, 1, 2` |
| What keyword immediately exits a loop?                                   | `break`   |

<img width="706" height="512" alt="Task 6" src="https://github.com/user-attachments/assets/8b6d2f16-f73b-421a-bb7a-28963f062999" />


## Task 7 – Conclusion

This room expanded on the basic Python concepts introduced in *Python: Simple Demo* and provided a foundation for developing more practical security scripts.

The room covered:

* Comments and terminal output with `print()`
* Variables and common data types
* Conditional statements and logical operators
* Type conversion
* F-strings
* Augmented assignment
* String length, indexing and slicing
* String methods and character checks
* Lists and dictionaries
* Arithmetic and membership operators
* `for` and `while` loops
* Numerical sequences with `range()`
* Loop control using `break` and `continue`

## Key takeaways

The most important lessons from the room were:

* `input()` returns a string unless the value is explicitly converted.
* The `type()` function reveals the data type of a value.
* F-strings make dynamic output easier to read and format.
* `=` assigns a value, while `==` compares two values.
* String and list slices include the start index but exclude the end index.
* Lists store ordered values.
* Dictionaries associate keys with values.
* The `.get()` method safely retrieves dictionary values when a key may be missing.
* `//`, `%` and `**` perform floor division, modulus and exponentiation.
* `in` and `not in` provide membership checks across several data structures.
* A `for` loop is suitable for iterating over a known sequence.
* A `while` loop is suitable when repetition depends on a changing condition.
* `break` terminates a loop.
* `continue` skips the remainder of the current iteration.

These concepts provide the foundation for writing Python tools that can process security-related data efficiently.

The next room, *Python: Building Scripts*, applies these concepts when creating more complete programs, including a password strength checker.

---

> All exercises documented here were completed in an authorised TryHackMe training environment.

