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

I started with a basic port scan and directory fuzzing.

- **Nmap** showed `22/tcp` (OpenSSH) and `80/tcp` (Go net/http server).  
- **Gobuster** discovered `/img`, `/r`, and `/poem`.  
- Inside `/img` I found images and used **StegSeek** — one image revealed a `hint.txt` with the phrase `follow the r a b b i t`, which pointed me to `/r/a/b/b/i/t`.

That steg hint and the folder structure nudged me to check the site source more closely — there I found a string that looked like credentials for `alice`.

---

## Initial access — `alice`

I used the discovered credentials to SSH in:

```text
ssh alice@<TARGET-IP>
# password: HowDothTheLittleCrocodileImproveHisShiningTail
