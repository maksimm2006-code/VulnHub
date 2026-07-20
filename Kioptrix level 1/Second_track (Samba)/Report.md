
Tools list:
	arp-scan
	nmap
	gobuster - Notting here
	searchsploit
	Metasploit


---

# 📝 Penetration Testing Report

# 🎯 VulnHub — Kioptrix Level 1

> **📌 Platform:** VulnHub  
> **🖥️ Target Machine:** Kioptrix Level 1  
> **💻 Attacking System:** Kali Linux  
> **🌐 Environment:** Oracle VirtualBox (Host-Only Network)

---

# 📖 Executive Summary

The objective of this assessment was to perform a penetration test against the **Kioptrix Level 1** virtual machine to identify vulnerable services and evaluate whether full system compromise was possible.

During the assessment, a known vulnerability in the **Samba** service was successfully exploited, resulting in remote access with **root-level privileges**.

---

# 💡 Skills Demonstrated

- 🌐 Network Reconnaissance
    
- 🔍 Service Enumeration
    
- 🏷️ Banner Grabbing
    
- 📂 SMB Enumeration
    
- 🔎 Vulnerability Research
    
- ✔️ Exploit Validation
    
- ⚙️ Metasploit Framework
    
- 🔄 Reverse Shell
    
- 👑 Post-Exploitation
    
- 📝 Security Reporting
    

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
🛰️ Host Discovery
        │
        ▼
🔍 Service Enumeration
        │
        ▼
🧩 Software Version Analysis
        │
        ▼
🔎 Known Vulnerability Research
        │
        ▼
✔️ Exploit Validation
        │
        ▼
🚀 Samba Exploitation
        │
        ▼
🔄 Reverse Shell
        │
        ▼
👑 Root Access
```

---

# 🛰️ 1. Network Reconnaissance

## 🎯 Objective

Identify the target system within the local network.

## 🤔 Why This Method

Since the lab environment was deployed inside an isolated VirtualBox Host-Only network, the first step was to identify active hosts.

```bash
sudo arp-scan -I eth1 192.168.96.0/24
```

An active host was discovered.

```text
192.168.96.15
```

### 🔎 Analysis

Once the target IP address was identified, the next step was to enumerate the exposed network services.

---

# 🔍 2. Service Enumeration

## 🎯 Objective

Identify exposed network services and determine their software versions.

## 🤔 Why This Method

Service versions provide valuable information for identifying publicly known vulnerabilities.

Aggressive Nmap scanning was performed to gather as much information as possible.

```bash
sudo nmap -sV -A 192.168.96.15
```

The following services were identified:

|🔌 Port|🛠️ Service|
|---|---|
|22|OpenSSH 2.9p2|
|80|Apache 1.3.20|
|111|RPCBind|
|139|Samba|
|443|Apache SSL|
|32768|RPC Status|

### 🔎 Analysis

Several services immediately stood out:

- ⚠️ Outdated Apache version
    
- ⚠️ SSHv1 support
    
- ⚠️ SSLv2 support
    
- ⚠️ Outdated Samba version
    

Since most of these services are no longer supported, the next step was to identify the most promising attack vector.

---

# 🌐 3. HTTP Enumeration

## 🎯 Objective

Assess the web application for accessible resources and information disclosure.

## 🤔 Why This Method

Web servers are often the primary entry point during penetration testing.

The main page was inspected first, but no useful information was found.

The page source was also reviewed, followed by directory enumeration.

```bash
gobuster dir \
-u http://192.168.96.15 \
-w /usr/share/wordlists/dirb/common.txt
```

Several directories were discovered; however, none contained information useful for further exploitation.

### 🔎 Analysis

HTTP enumeration did not reveal any viable attack vectors.

As a result, the assessment shifted focus to other exposed services.

---

# 📂 4. SMB Enumeration

## 🎯 Objective

Determine the exact version of the Samba service.

## 🤔 Why This Method

Nmap did not identify the Samba version during the initial scan.

To obtain more accurate information, the following Metasploit auxiliary module was used:

```text
auxiliary/scanner/smb/smb_version
```

The service was identified as:

```text
Samba 2.2.1a
```

### 🔎 Analysis

The identified Samba version is significantly outdated.

Instead of immediately attempting exploitation, the next step was to verify whether publicly known vulnerabilities existed for this version.

---

# 🔎 5. Vulnerability Research

## 🎯 Objective

Confirm the existence of a working exploit for the identified Samba version.

## 🤔 Why This Method

Exploitation should always be based on validated information rather than assumptions.

Public security resources identified the following vulnerability:

```text
trans2open
```

The finding was then verified using Searchsploit.

```bash
searchsploit samba 2.2.1
```

A matching exploit was found.

The available Metasploit modules were also reviewed.

```text
search trans2open
```

The following exploit module was identified:

```text
exploit/linux/samba/trans2open
```

### 🔎 Analysis

Only after confirming the availability of a suitable exploit was the decision made to proceed with exploitation.

This approach minimizes the risk of using incompatible or unreliable exploits.

---

# 🚀 6. Exploitation

## 🎯 Objective

Gain remote access to the target system.

## 🤔 Why This Method

The following conditions had already been verified:

- ✔️ Samba version
    
- ✔️ Presence of a known vulnerability
    
- ✔️ Availability of a compatible exploit
    

The following payload was selected:

```text
generic/shell_reverse_tcp
```

The exploit was then executed.

A reverse shell was successfully obtained.

To verify the level of access, the following commands were executed:

```bash
whoami

cat /etc/shadow
```

The results confirmed that the obtained shell already had **root privileges**.

### 🔎 Analysis

The exploitation was successful due to a well-known critical vulnerability affecting the outdated Samba version.

No privilege escalation was required.

---

# 👑 7. Post-Exploitation

## 🎯 Objective

Confirm full control over the target system.

After obtaining shell access, the filesystem was examined.

The initial search attempted to locate a file named:

```bash
find / -name "flag"
```

No matching file was found.

Further manual enumeration eventually located the file containing the challenge flag.

### 🔎 Analysis

At this stage, the objective of the assessment was considered successfully completed, as full administrative control over the target system had been achieved.

---

# ⚠️ Findings

|⚠️ Finding|📊 Risk|
|---|---|
|Outdated Samba 2.2.1a|🔴 Critical|
|SSHv1 Enabled|🟠 High|
|SSLv2 Enabled|🟠 High|
|Outdated Apache Version|🟠 High|
|TRACE Method Enabled|🟡 Medium|

---

# ✅ Conclusion

This laboratory demonstrates the security risks associated with running outdated software.

Rather than immediately launching an exploit, the assessment followed a structured methodology:

- 🔍 Identifying service versions;
    
- 📚 Researching publicly disclosed vulnerabilities;
    
- ✔️ Verifying the availability of a compatible exploit;
    
- 🚀 Proceeding with exploitation only after validation.
    

This project reflects my penetration testing methodology: **first gather as much information as possible about the target system, validate the applicability of known vulnerabilities, and only then proceed with exploitation.**
