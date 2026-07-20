
My IP Adress : 192.168.96.3

Target IP Adress: 192.168.96.15


---

Todo List:
	arp-scan
	nmap
	nikto
	searchsploit


# 📝 Penetration Testing Report

# 🎯 VulnHub — Kioptrix Level 1

> **📌 Platform:** VulnHub  
> **🖥️ Target Machine:** Kioptrix Level 1  
> **💻 Attacking System:** Kali Linux  
> **🌐 Environment:** Oracle VirtualBox (Host-Only Network)  
> **🎯 Objective:** Achieve full system compromise by analyzing the attack surface and exploiting identified vulnerabilities.

---

# 📌 Executive Summary

The objective of this assessment was to perform a complete penetration test against the **Kioptrix Level 1** virtual machine in order to identify vulnerable services and evaluate whether full system compromise was possible.

During the assessment, a critically outdated **Apache + mod_ssl + OpenSSL** stack was identified. Further investigation confirmed the presence of a publicly known remote code execution vulnerability. After validating the availability of a suitable public exploit, remote code execution was successfully achieved, resulting in a root shell on the target system.

---

# 💡 Skills Demonstrated

- Network Reconnaissance
    
- Service Enumeration
    
- Banner Analysis
    
- Web Enumeration
    
- Vulnerability Assessment
    
- Searchsploit
    
- Public Exploit Research
    
- Exploit Compilation
    
- Remote Code Execution (RCE)
    
- Post-Exploitation
    
- Security Reporting
    

---

# 🖥️ Lab Architecture

```text
Kali Linux
      │
      │ Host-Only Network
      │
Kioptrix Level 1
```

---

# 🛣️ Attack Path

```text
Host Discovery
        │
        ▼
Service Enumeration
        │
        ▼
Apache Banner Analysis
        │
        ▼
Nikto Scan
        │
        ▼
Known CVE Research
        │
        ▼
Searchsploit
        │
        ▼
GitHub Public Exploit
        │
        ▼
Exploit Compilation
        │
        ▼
Remote Code Execution
        │
        ▼
Root Shell
```

---

# 🛰️ 1. Network Reconnaissance

## 🎯 Objective

Identify the IP address of the target machine.

## 🤔 Why This Method

Since the lab environment was deployed within an isolated VirtualBox Host-Only network, the first step was to identify active hosts.

```bash
sudo arp-scan -I eth1 192.168.96.0/24
```

An active host was discovered.

```text
192.168.96.15
```

### 🔎 Analysis

After identifying the target, the next logical step was to enumerate its exposed attack surface.

---

# 🔍 2. Service Enumeration

## 🎯 Objective

Identify exposed network services and determine their software versions.

## 🤔 Why This Method

Accurate service versions are essential for identifying publicly known vulnerabilities.

Aggressive Nmap scanning was used to gather comprehensive information.

```bash
sudo nmap -A -sV 192.168.96.15
```

The following services were identified:

|🔌 Port|🛠️ Service|
|---|---|
|22|OpenSSH 2.9p2|
|80|Apache 1.3.20|
|111|RPCBind|
|139|Samba|
|443|Apache SSL|
|32768|RPC|

The following Apache banner immediately attracted attention:

```text
Apache/1.3.20 (Unix)
mod_ssl/2.8.4
OpenSSL/0.9.6b
```

### 🔎 Analysis

All components of the web server stack were significantly outdated.

Knowing the exact software versions allowed further research into publicly disclosed vulnerabilities.

---

# 🌐 3. Web Server Analysis

## 🎯 Objective

Assess the web server for known vulnerabilities.

## 🤔 Why This Method

Banner information alone is insufficient to justify exploitation.

Nikto was used to perform an additional security assessment.

```bash
nikto -h http://192.168.96.15
```

Nikto reported the following finding:

```text
mod_ssl 2.8.7 and lower are vulnerable to a remote buffer overflow which may allow a remote shell.
```

### 🔎 Analysis

Nikto confirmed the presence of a well-known critical vulnerability affecting the installed version of **mod_ssl**.

Before proceeding, it was necessary to verify that a compatible public exploit existed.

---

# 🔍 4. Public Exploit Research

## 🎯 Objective

Validate that the identified vulnerability was practically exploitable.

## 🤔 Why This Method

Before using any exploit, it is important to confirm that it matches the exact software version running on the target.

Research of public sources identified the **OpenFuck (OpenFuckV2)** exploit.

Searchsploit was then used for verification.

```bash
searchsploit mod_ssl
```

Searchsploit confirmed a matching exploit.

```text
Apache mod_ssl < 2.8.7 OpenSSL
OpenFuck.c
```

A maintained implementation of the exploit was subsequently located on GitHub.

### 🔎 Analysis

At this stage, exploitation feasibility had been confirmed.

The next step was preparing and compiling the exploit.

---

# 🚀 5. Exploitation

## 🎯 Objective

Obtain remote code execution on the target.

## 🤔 Why This Method

The OpenFuck exploit requires selecting the correct target profile.

The previously collected Apache banner was used to determine the appropriate target identifier.

```text
Apache/1.3.20 (Unix)
```

The exploit was then executed.

```bash
./OpenFuck 0x6b 192.168.96.15 443 -c 50
```

The attack completed successfully.

A remote shell session was established.

### 🔎 Analysis

Remote code execution was achieved by exploiting the vulnerable **mod_ssl** implementation.

It is worth noting that the **0x6b** target identifier was selected based on evidence collected during service enumeration rather than by trial and error.

---

# 👑 6. Post-Exploitation

## 🎯 Objective

Confirm full control over the target system.

After obtaining shell access, the current user was identified.

```bash
whoami
```

The filesystem was then inspected.

An initial search for a flag file was performed.

```bash
find / -name "flag"
```

No matching file was found.

Further manual enumeration of the filesystem revealed the file confirming successful completion of the laboratory machine.

### 🔎 Analysis

The obtained shell already had **root privileges**, making privilege escalation unnecessary.

---

# ⚠️ Findings

|⚠️ Finding|📊 Risk|
|---|---|
|Outdated Apache version|🟠 High|
|Vulnerable mod_ssl version|🔴 Critical|
|OpenSSL 0.9.6b|🔴 Critical|
|SSLv2 enabled|🟠 High|
|SSHv1 supported|🟠 High|

---

# ✅ Conclusion

The compromise of the target system was made possible by the use of critically outdated web server components.

The assessment followed a structured methodology:

- 🔍 Identifying software versions;
    
- 🧩 Analyzing service banners;
    
- 🛠️ Verifying findings with specialized security tools;
    
- 📚 Researching publicly disclosed vulnerabilities;
    
- ✔️ Confirming the availability of a working exploit;
    
- 🚀 Proceeding with exploitation only after validating applicability.
    

This project demonstrates my penetration testing methodology: **every action is driven by evidence gathered during the previous phase, and exploitation is performed only after confirming that a vulnerability is applicable to the target's specific configuration.**
