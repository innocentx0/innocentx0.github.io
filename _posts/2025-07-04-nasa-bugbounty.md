---
title: "How I Hacked NASA and Scored a Killer LOR 🚀"
date: 2025-07-19 00:00:00 +0800
categories: [Bug bounty, hacking]
tags: [Bug bounty, NASA Hacking, Hacking]
--- 

# How I Hacked NASA and Scored a Killer LOR 🚀👨‍💻

Today, I’m going to tell you an insane story — a legendary hacker tale: **how I hacked NASA**. Yep, you heard that right. The NASA. The space agency that sends rovers to Mars and makes us dream of the stars? They got owned by me. And the cherry on top? I got a **Letter of Recognition (LOR)** straight from them.  
But trust me, behind this epic tale lies weeks of madness — recon, scripts, insane bugs, and a whole lot of craziness.

---

## 🐛 My NASA Bug Hunt: A Dozen Mind-Blowing Vulnerabilities

I found around **ten bugs** in their systems — stuff straight out of a movie. I’m talking about:

- Accessing **super sensitive APIs**
- Leaking **locations of all NASA bases**
- Employee personal data (**PII included**)
- **Classified documents**, top-secret research marked as **reserved**

One of the craziest bugs?  
A NASA site protected by **Basic Auth** — I bypassed it just by **tweaking the HTTP request header**.  
No brute force. No magic. One crafted header = full access.

---

## 🏆 The Bug That Earned Me the LOR

This deserves its own spotlight.

I spent **a week** preparing the attack, days of recon, endless data review — until one day, buried in **8 GB of scraped data**, I found *the link*.  
The one that wasn’t supposed to exist.

---

## 🔍 How I Started: Recon Mode Activated

### 1. Subdomain Enumeration

Used:
- [`crt.sh`](https://crt.sh/)
- `subfinder`
- `assetsfinder`

Found **dozens** of subdomains.

### 2. Liveness Check

Used `httpx-toolkit` to check which domains were still active.

### 3. Dual-Method Link Scraping

- **Active method:**  
  My own spider, `Reconspyder`, built with `Scrapy`, did:
  - Directory fuzzing
  - Page enumeration
  - Analysis via `GAU`, `linkfinder`

- **Passive method:**  
  The secret sauce: **WayBackMachine**.

---

## 📜 What Is the WayBackMachine (And Why Hackers Should Use It)

The **WayBackMachine** is basically a **time machine for the web**.  
It archives snapshots of websites across time.  
For hackers, it’s a **goldmine**:  

- See **old versions** of pages  
- Access **deleted files**  
- Find **once-exposed directories**  
- Discover endpoints that devs forgot  

Using a custom script, I pulled **every archived link** from each domain.  
The result: **~10 GB of files**.

---

## 🧪 File Scanning and Thinking Outside the Box

I wrote a script to **grep everything**, targeting:

- `password`, `user`, `admin`
- `ID=` and other sensitive parameters
- `redirect=`, `next=`, possible open redirects
- `/admin`, `/login`, `/config`
- API keys
- SSH keys
- `.env` files
- And anything that screamed **vuln**

After 2 days, I had a thought:  
> "What would a NASA scientist use to share internal data?"

---

## 🧠 The Big Brain Moment: FTP Discovery

Yup: **FTP**.

I grepped for `ftp`, and I found this gem:

---

## 🚫 The Forbidden Page

The page displayed this warning:

> “You are not authorized to access this content. This section is reserved exclusively for NASA employees. Unauthorized access is strictly prohibited and punishable under federal laws.”

Classic.

But I wasn’t done.

---

## 🧵 Down the Rabbit Hole: Cached Treasures

Digging deeper into WayBackMachine, I uncovered **tons of cached links**:

- Ongoing NASA **research projects**
- **3D models** of satellites and rovers
- **Geospatial datasets**
- Confidential **scientific reports**
- Internal **project documentation**
- Source code of **proprietary tools**

All just sitting there. Waiting to be seen.

---

## 📬 Reporting the Bug and Receiving the LOR

After gathering all my findings, I submitted a clean and detailed report to NASA’s bug bounty team.

- **Severity:** P3  
- **Impact:** High  
- **Result:** An **official Letter of Recognition** from the agency.

Mission accomplished. ✅

---

## 🧠 Final Thoughts

This adventure was a perfect blend of:

- Curiosity  
- Lateral thinking  
- Automation  
- Attention to detail  
- A pinch of madness

### 🔑 Key Lessons:
- Recon is king  
- The **WayBackMachine** is an underrated weapon  
- Think like the target  
- And always, **always** document everything  

---

## 🚀 To the Stars (and Bugs) Beyond

Now? I still watch the stars…  
But with my keyboard ready for the next space-borne vulnerability.

Happy hacking. 💻🌌

---


