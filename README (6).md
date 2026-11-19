# DVWA Exploitation Write-up — SQL Injection, XSS & Sensitive File Disclosure

A hands-on penetration testing exercise against **DVWA (Damn Vulnerable Web Application)**, running locally on a Kali Linux VM. This write-up moves from vulnerability identification to **controlled exploitation**, with evidence captured at every step using Burp Suite, curl, and the browser.

Built as **Task 4** of the Cybersecurity & Ethical Hacking Internship at Maincrafts Technology (follows on from the Task 3 Vulnerability Assessment).

## Scope

All testing was performed exclusively against DVWA, running on an isolated local Kali VM. No live, public, or third-party system was accessed or tested.

## What's inside

| Vulnerability | Location | Risk |
|---|---|---|
| SQL Injection | SQL Injection module (User ID field) | Critical |
| Cross-Site Scripting (Reflected & Stored) | XSS (Reflected) URL parameter; XSS (Stored) guestbook Message field | High |
| Exposed configuration backup file | `/config/config.inc.php.bak` | High |

## 1. SQL Injection

**Payload:**
```
1' OR '1'='1
```
Bypasses the query's WHERE clause, returning every row instead of one.

**Data extraction (UNION-based):**
```
1' UNION SELECT user, password FROM users -- 
1' UNION SELECT database(), version() -- 
```
Pulls usernames and password hashes directly from the `users` table, and fingerprints the backend as MariaDB 10.1.26.

![SQLi bypass](screenshots/01_sqli_or1eq1.png)
![SQLi extracted hashes](screenshots/02_sqli_union_hashes.png)

**Why it works:** the User ID field is concatenated directly into the SQL query with no sanitisation or parameterisation, so closing the quote early and appending a boolean condition — or a `UNION SELECT` with a matching column count — lets an attacker control what the query returns.

## 2. Cross-Site Scripting (XSS)

**Payload:**
```html
<script>alert('XSS')</script>
```

- **Reflected** — injected via the URL's `name` parameter, executes immediately.
- **Stored** — saved via the guestbook's Message field, executes for every visitor who views the page, and persists across reloads (confirmed via the browser's form-resubmission prompt).

![Reflected XSS](screenshots/03_xss_reflected.png)
![Stored XSS](screenshots/04_xss_stored.png)

**Why it works:** user input is rendered into the HTML response without encoding `<` and `>`, so a submitted `<script>` tag is parsed and executed by the browser instead of displayed as text.

## 3. Sensitive File Disclosure

A backup of DVWA's own configuration file was found publicly accessible and retrieved with a single request:

```bash
curl http://localhost/config/config.inc.php.bak
```

This discloses the live database host, name, username, and password in plain text — no exploitation of any other vulnerability needed.

![Config file disclosure](screenshots/05_config_bak_disclosure.png)

## 4. Burp Suite

Used to intercept and manually replay requests outside the browser, confirming the SQL Injection could be triggered with full control over the raw HTTP request.

![Burp Proxy intercept](screenshots/06_burp_proxy.png)
![Burp Repeater](screenshots/07_burp_repeater.png)

## Fix recommendations

- Use parameterised queries / prepared statements instead of string concatenation.
- Validate and encode all user input (expect an integer for `id`, encode output before rendering).
- Store passwords with a salted, adaptive hash (bcrypt/Argon2) instead of unsalted MD5.
- Remove backup files (`.bak`, `.old`, `.orig`) from web-accessible directories as part of the deployment process.

## Full report

See [`Cybersecurity_Ethical_Hacking_Task4_Report.docx`](Cybersecurity_Ethical_Hacking_Task4_Report.docx) for the complete write-up, including the OWASP Top 10 concept document, full evidence set, and a consolidated end-to-end mini penetration test.

## Responsible use

All testing was performed only against an intentionally vulnerable application in an isolated lab environment. Do not run these techniques against systems you do not own or have explicit written permission to test.
