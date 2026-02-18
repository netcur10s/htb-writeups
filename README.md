# 🟢 Hack The Box Writeups

These writeups are structured to go beyond just "how I got the flag" — 
each one includes detection opportunities and defensive lessons learned, 
reflecting the dual awareness required in SOC work. All machines are 
retired before writeups are published, in line with HTB's rules.

## 📂 Writeups

### Linux

| Machine | Difficulty | Techniques | Status |
|---------|------------|------------|--------|
| [Code Part Two](/codeparttwo.md) | Easy | CVE-2024-28397, Hash Cracking, Sudo Abuse | ✅ Complete |
| [Cap](/cap.md) | Easy | IDOR, PCAP Analysis, Credential Extraction, Linux Capabilities (cap_setuid) | ✅ Complete |

### Windows

| Machine | Difficulty | Techniques | Status |
|---------|------------|------------|--------|
| Coming soon | — | — | 🔄 In Progress |

## 📊 Progress

![Machines Completed](https://img.shields.io/badge/Machines%20Completed-2-brightgreen)
![Linux](https://img.shields.io/badge/Linux-2-blue)
![Windows](https://img.shields.io/badge/Windows-0-lightgrey)

## 📝 Writeup Structure

Each writeup follows this format:
- **Machine Info** — OS, IP, difficulty
- **Summary** — Attack chain overview
- **Enumeration** — Recon and service discovery
- **Initial Access** — How the foothold was gained
- **Lateral Movement** — Pivoting between users (if applicable)
- **Privilege Escalation** — Path to root/admin
- **Key Takeaways** — Defensive lessons from each vulnerability
- **References** — CVEs and MITRE ATT&CK mappings

## 🛠️ Common Tools

- Nmap, Gobuster, Burp Suite
- Metasploit, Impacket
- Hashcat, John the Ripper
- Wireshark, tcpdump

## 📫 Connect

- **LinkedIn:** [linkedin.com/in/vic1101](https://linkedin.com/in/vic1101)
- **Email:** v.echevarria@proton.me
