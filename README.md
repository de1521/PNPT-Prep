# PNPT Prep

My own notes for the PNPT (Practical Network Penetration Tester). I'm making a career move into offensive security and I kept rebuilding the same commands from memory every time I sat down to practice, so I put them all in one place — ordered the way you actually use them during a pentest, not alphabetically.

I learn by doing, not by reading, so this isn't a tutorial. It's a fast reference: find the phase you're stuck on, grab the command, get back to the box.

## What's in here

- **`pnpt-methodology.md`** — the full attack chain, phase by phase, with the commands I reach for and the mistakes I kept making (they're marked with ⚠).
- **`cheatsheets/`** — deeper notes on the parts that gave me the most trouble: moving files between Kali and Windows, and the Windows credential attacks (SAM, hashes, pass-the-hash).
- **Interactive map** — the same methodology as a webpage you can search and click through.  https://de1521.github.io/PNPT-Prep/
- - **Windows PrivEsc Attack Map** — an interactive map of Windows privilege-escalation techniques: https://de1521.github.io/PNPT-Prep/windows-privesc-map.html
  - - **Rogue Potato Attack Map** — SeImpersonate → SYSTEM, step by step: https://de1521.github.io/PNPT-Prep/rogue-potato-map.html
    - - **PrintSpoofer Attack Map** — SeImpersonate → SYSTEM via the Print Spooler: https://de1521.github.io/PNPT-Prep/printspoofer-map.html
       - - **Wreath Webserver Attack Map** — recon → exploit → persistent root on the Wreath webserver (CVE-2019-15107): https://de1521.github.io/PNPT-Prep/wreath-webserver-map.html
         - - **Pivoting Map** — interactive reference for tunnelling through a compromised machine (ssh -L/-R/-D, proxychains, socat, chisel): https://de1521.github.io/PNPT-Prep/pivoting-map.html
           - - **pivoting.md** — the pivoting toolkit: enumeration, SSH tunnels, plink, socat (incl. OpenSSL-encrypted pipes), and chisel — with the when & why for each.
         -  
## Who this is for-

Mostly me. But if you're also grinding toward the PNPT and you think in terms of "what do I run next," it might save you some time too. Corrections welcome.

## A note on the exam

Everything here is general methodology and public tools — the stuff that's on TryHackMe, in the TCM courses, and in every pentest. **Nothing in here is about the real PNPT exam** (network, machines, flags). That's under NDA and stays that way.

## Still a work in progress

Same as me. I add to it after every study session.


Credits

The notes, methodology, and the mistakes are my own, from doing the labs myself. I used AI assistance to help format and build these documents and the interactive map.
