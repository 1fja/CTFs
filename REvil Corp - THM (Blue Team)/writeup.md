# REvil Corp - Write Up

**Note: This write-up documents an investigation performed in a virtual machine hosted by TryHackMe. The environment was intentionally designed for security training. The images presented in this write-up have been obtained by other write-ups (because I forgot to screenshot). But all the steps and investigation was conducted by me, being loyal to what I did.**

# Executive Summary
| Category   | Value                                                                                                        |
| ---------- | ------------------------------------------------------------------------------------------------------------ |
| Platform   | TryHackMe                                                                                                    |
| Challenge  | REvil Corp                                                                                                   |
| Difficulty | Medium                                                                                                       |
| Type       | Digital Forensics & Incident Response                                                                        |
| Tool Used  | Mandiant Redline                                                                                             |
| Objective  | Investigate a ransomware infection and identify attacker artifacts, victim activity, and malware indicators. |

A employee at Lockman Group reported that his files got renamed with to a strange extension, the IT department classified such behavior as a ransomware activity and escalated the case to the Incident Response team. The investigation revealed several artifacts associated with the ransomware infection, including suspicious downloads, encrypted files, a modified desktop wallpaper, and a ransom note. These findings helped identify the malware and better understand its impact on the victim's system.

# Investigation

The first step of the investigation started by seeing the system information
<img width="828" height="615" alt="image" src="https://github.com/user-attachments/assets/3689cf4a-1e17-496e-8d52-08d894e5758f" />

The operating system present in the victim's machine was a Windows 7, nowadays can be classfied as a Vulnerable and Outdated components (A06:2021) according to OWASP, and such operating system have known vulnerabilities that can compromise the whole system, this could lead to other attacks and could even contributed to the ransomware attack. Besides that, we can also confirm the victim's name by going to the system information section.


Now that we've confirmed what user and operating system the victim has, let's see what file has compromised the machine by going in the file download history section.
<img width="828" height="336" alt="image" src="https://github.com/user-attachments/assets/4eb6855a-c5be-44e1-a9fb-0b853572bd2a" />

<img width="720" height="58" alt="image" src="https://github.com/user-attachments/assets/a27183fc-1ca3-47b2-b839-f373674dcab7" />

Analysing these two URLs/downloads, we can assume that the user downloaded the WINRAR from an untrusted source. The source had also instructed the victim to download the TOR Browser to actually get the desired software.

This is very common in youtube tutorials or forums that trick the user into downloading a malicious file, this social engineering behavior has the solely purpose to gain financial advantage of the victim. The incident could've been prevented by a cyber awareness training.

Now, the MD5 hash of this file has to be gathered to further investigation later in this machine. By going to the File System section, filtering the users and searching for the malicious file, we can obtain the MD5 hash.

<img width="828" height="164" alt="image" src="https://github.com/user-attachments/assets/2ba94d0d-bc2b-4d3d-a2db-650dbcd7a36c" />

<img width="669" height="445" alt="image" src="https://github.com/user-attachments/assets/e37601b1-b8f6-40b3-80f9-bbf2373a00f5" />

If we search the MD5 hash, we can have more details about the ransomware. Revealing the name 'REvil' and using anyrun, we can view the notes, wallpaper, payloads and behaviour.

Reading the note located in the desktop, the note says that the user should not download any other tools to try and decrypt the files and some other information

<img width="1915" height="934" alt="image" src="https://github.com/user-attachments/assets/317e73ef-ea11-48b1-941e-ee008d2e6e7b" />

<img width="1035" height="626" alt="image" src="https://github.com/user-attachments/assets/5a2d160a-e2ca-4685-90f0-c116f74eab91" />


We can also take notice of the file size, which is 164KB


Let's now see what file extension the ransomware has given to the other files.

<img width="720" height="659" alt="image" src="https://github.com/user-attachments/assets/716e83f0-ec55-4ef7-bc19-db1654d1f8af" />

We can already observe that the odd extension that the victim has reported is 't48s39la'. Now, we should investigate the number of files that have been changed in this incident.

By going to the Timeline section, selecting only the boxes 'Modified' and 'Changed' and search for the extesion, we can obtain the exact ammount of files that got encrypted.

<img width="828" height="517" alt="image" src="https://github.com/user-attachments/assets/87100013-6545-49c3-9913-0a466ce85527" />

Further investigation revealed that the victim's wallpaper was modified to a .bmp file, filtering this in the File System or Timeline section, we can see the modified wallpaper

<img width="483" height="267" alt="image" src="https://github.com/user-attachments/assets/607d927d-a449-4a73-a586-24ef6f0a9a93" />

Also, the ransomware left a note in the victim's desktop

<img width="828" height="458" alt="image" src="https://github.com/user-attachments/assets/3f31fe80-3145-4e9a-af42-46e3643939b6" />

The ransomware has also changed an official URL in the favorited directory of the victim's machine, the 'GobiernoUSA' url. By searching this URL, it revealed that that was actually a real domain of the USA governament that had the purpose to combat desinformation, health advice and general support in natural disasters, but this domain was discontinued in 2017 and got replaced by 'usa.gov/es'.

This change was frequently abused to gain local persistence, which is very common in malwares.

<img width="828" height="395" alt="image" src="https://github.com/user-attachments/assets/e7d01069-083e-4fa0-b0cd-9ade458dbb0f" />

Now, there's a file that has 0KB of size inside of the desktop

<img width="720" height="454" alt="image" src="https://github.com/user-attachments/assets/3b7dd290-ccb3-4597-95fb-88fa0e3cbcaf" />

We can also observe that the victim had a desperate attempt to decrypt the file, that was controlled by the authors of that ransomware
<img width="709" height="93" alt="image" src="https://github.com/user-attachments/assets/612d9362-a615-4d51-adfd-9565cc3bfb0d" />
<img width="653" height="310" alt="image" src="https://github.com/user-attachments/assets/357740e7-0a20-492e-9d62-8123c5d26e68" />

Let's investigate the source of this decryptor file, by going in the URL history section.

<img width="661" height="431" alt="image" src="https://github.com/user-attachments/assets/9d14a855-073a-44de-ad89-ddc6faf72c85" />

and searching the REvil ransomware names, we get:
revil,sodin,sodinokibi

# Personal Reflection

The REvil Corp room was an interesting environment to investigate. It helped me gain more experience with Blue Team operations, and I found the investigation process genuinely enjoyable.

One of my biggest takeaways was the importance of using file hashes to gather additional information about suspicious files. I had initially overlooked this approach, but it proved useful during the investigation.

Surprisingly, identifying the modified wallpaper was one of the most challenging parts for me. I expected it to have a familiar file extension, so I did not initially consider the possibility of a different one. This reminded me to avoid assumptions and examine artifacts more carefully during forensic investigations.
