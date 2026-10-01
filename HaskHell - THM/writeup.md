# HaskHell - TryHackMe Write Up

**Platform:** TryHackMe  
**Target:** HaskHell  
**Operating System:** Linux

 **Note:** This write-up documents a penetration test performed against a virtual machine hosted by TryHackMe. The environment was intentionally designed for security training.

---

# Enumeration

I started by scanning all TCP ports on the target:

```
nmap -p- -sCV <TARGET_IP> --min-rate 1000
```
The scan revealed two interesting services:
```
22/tcp   open  ssh
5001/tcp open  http  Gunicorn 19.7.1
```
The HTTP service was running Gunicorn and had a page titled:

``Homepage``

Inside the landing page (or homepage) there was a note from the teacher saying on how to upload the code of the exercise that he was asking for, but clicking on the button to submmit, it didn't work

So I started enumerating its directories.

# Web Enumeration

I used dirsearch against port 5001:
```
dirsearch -x 404 -e -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt --url http://<TARGET_IP>:5001/ -t 100
```
Among the results, two paths immediately stood out:
```
/submit
/uploads/affwp-debug.log
```
The ``/submit`` endpoint was particularly interesting because it allowed files to be submitted to the application and that's the attacking vector we're looking for.

# Initial Access

The /submit is the endpoint that accepts Haskell files.

Instead of first uploading a harmless Haskell script just to verify whether the server would execute it, I went directly for the more useful hypothesis:

- If the server accepts and executes Haskell code, I should be able to use it to obtain a reverse shell.

I prepared a Haskell reverse-shell payload (obtained in revshells.com) and submitted it through the endpoint.

The payload executed successfully and I received a reverse shell on my listener.

This gave me my initial foothold on the machine and the first flag

# User Enumeration

At this point, I was operating as the low-privileged user flesk.

I investigated the usual privilege-escalation possibilities, but I couldn't find a useful escalation path from this account.

Instead of spending too much time forcing a privilege-escalation vector that wasn't immediately apparent, I looked for another way to move laterally. So that's when I've decided to change to another home directory and I found the home directory for 'prof'

### Moving to prof

I was able to obtain access to this account and SSH into the machine as prof. Because apparently, the directory had misconfigured permissions and I was able to read the ``id_rsa`` key

This changed the situation because prof had access to additional functionality that wasn't available from the previous account.

# Privilege Escalation

While enumerating the prof environment (and for common privilege escalation paths), I found an interesting environment variable:

``flask_app``

The value of this variable pointed toward functionality that could be abused to execute code with higher privileges.

I investigated how the application was using this environment variable and identified a path toward command execution and then created a Python reverse-shell script and used the discovered behavior to execute it.

I've setup a listener and I was able to make a reverse shell connecting back to my listener with elevated privileges.

I confirmed the result with:

``whoami``

The result was:

``root``

At this point, the machine was fully compromised and I was able to get the root flag.

# Final Thoughts

HaskHell was an interesting and easy machine because the initial foothold was relatively direct once the /submit endpoint was discovered.

The important part for me was recognizing that the application accepted Haskell code and immediately considering code execution as the objective, rather than spending unnecessary time testing the upload with a harmless script first.

After gaining the initial shell, I couldn't find a useful privilege escalation path as flesk, so I changed direction and investigated other users. This eventually led me to prof and, from there, to the flask_app environment variable that provided the privilege-escalation path.

A lesson I would give to beginners from this machine is:

Don't get attached to your first foothold. If the current user doesn't give you a clear path forward, keep enumerating and look for another path through the system.
