# Windows PrivEsc — Pass-the-Hash (TryHackMe, Task 12)

A writeup of the Pass-the-Hash task from the TryHackMe **Windows PrivEsc** room (the Tib3rius one). This one follows straight on from the SAM attack — once you've got a hash, you don't even need to crack it.

Everything here is a public practice box and generic methodology. Nothing about the real PNPT exam (NDA).

## Summary

- **Box:** TryHackMe — Windows PrivEsc room, Task 12 (Passing the Hash)
- **Start:** I already have the admin account's NTLM hash (dumped from the SAM in Task 11)
- **End:** a SYSTEM shell on the target — logged in using only the hash, no password cracked
- **The idea:** authenticate to Windows with the stolen hash directly

## The concept — why this works

When Windows checks a login over the network (NTLM), it never compares the actual *password*. It proves you know the password by checking you have the **hash**. So the hash itself is the credential.

That means: if you've stolen the hash, you can log in **without ever knowing or cracking the password** — you hand Windows the hash, and it accepts it as proof.

Important: a hash is **one-way** — you never "decrypt" it. Cracking (Task 11) is just guessing passwords until one produces a matching hash. Pass-the-Hash skips that entirely — you don't need the password at all, just the hash.

## One thing that trips people up: full hash vs NT hash

- **hashcat** (cracking) wants **only the NT hash** — the 4th field from secretsdump.
- **Pass-the-Hash** wants the **full hash — LM:NT together**, joined by a colon:
  `aad3b435b51404eeaad3b435b51404ee:<NThash>`

The `aad3b435...` LM part is the dead/empty LM hash (same on every modern Windows account), but Pass-the-Hash still wants the whole `LM:NT` string.

## Exploitation

The room shows this with `pth-winexe`:

```bash
pth-winexe -U 'admin%<LMhash>:<NThash>' //TARGET cmd.exe
```

`pth-winexe` wasn't installed on my attacker box, so I used Impacket's psexec instead — same job, passes the hash the same way:

```bash
psexec.py -hashes aad3b435b51404eeaad3b435b51404ee:<NThash> admin@TARGET_IP
```

- `psexec.py` = Impacket's PsExec
- `-hashes LM:NT` = pass the full hash instead of a password
- `admin@TARGET_IP` = log in as the admin account on the target

Then:

```cmd
whoami
```

Returned `nt authority\system`. Logged in as SYSTEM, no password, no cracking.

## Mistakes / notes (so you can skip them)

1. **`pth-winexe: command not found`** — it wasn't installed, and `apt install winexe` couldn't find it either. Don't waste time installing — use what's there. Impacket's `psexec.py -hashes` does the same thing. (Same lesson as `winexe` and `impacket-smbserver` in earlier tasks: when the named tool is missing, reach for Impacket.)
2. **Don't confuse the hash formats** — hashcat = NT hash only; Pass-the-Hash = full `LM:NT`. Feed the wrong one and it fails.
3. **A hash is not decrypted** — it's one-way. You either crack it (guess) or pass it (reuse). You never reverse it.

## Impact

NTLM authentication accepts the hash as proof of identity, so a dumped hash can be reused directly — no cracking needed. If the same local admin password (and therefore the same hash) is used across many machines, one stolen hash opens all of them. That's the engine of **lateral movement**: pass the hash machine to machine, pivot across the network, and ride it up to Domain Admin.

## Remediation

- **Unique local admin passwords per machine** so one stolen hash only opens one box. Microsoft **LAPS** does this automatically (random password per machine).
- **Credential Guard** to make stealing the hash harder in the first place.
- Limit who has local admin rights, and restrict lateral movement (host firewall rules, disabling unused admin shares).
