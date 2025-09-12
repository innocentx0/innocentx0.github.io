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

![Nmap scan](/assets/blackfield/nmap-scan.jpg)



