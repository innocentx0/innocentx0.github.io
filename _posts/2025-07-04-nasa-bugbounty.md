---
title: "How I Got a Letter of Recognition from NASA"
date: 2025-07-19 00:00:00 +0800
categories: [Bug bounty, hacking]
tags: [Bug bounty, NASA, Hacking]
---

# How I Got a Letter of Recognition from NASA

A while back I found a handful of bugs in NASA's systems and ended up with a Letter of Recognition (LOR) from them. It took a few weeks of recon, a lot of scripting and way too much coffee. Here's how it went.

## The bugs

I reported around ten issues in total. Some of the things I could reach:

- sensitive APIs
- the locations of NASA facilities
- employee personal data, PII included
- internal documents marked as reserved

The simplest one was a site protected by Basic Auth. I got past it just by changing a header in the HTTP request. No brute force, nothing fancy.

The one that got me the LOR was different, and it took a lot more work.

## Recon

I spent about a week on this before touching anything. Most of it was reading data.

**Subdomains.** I used crt.sh, subfinder and assetfinder. The list was huge.

**Live hosts.** I ran everything through httpx-toolkit to see what was still up.

**Links.** I did this in two ways. For the active part I used my own Scrapy-based tool, Reconspyder, plus directory fuzzing, page enumeration, gau and linkfinder. For the passive part I used the Wayback Machine.

## Why the Wayback Machine

It stores snapshots of sites over time, which means you can see old versions of pages, files that have since been deleted, directories that used to be exposed and endpoints the developers forgot about.

I wrote a script to pull every archived URL for each domain. It came out to roughly 10 GB of data.

## Going through the data

I grepped for the usual suspects: `password`, `user`, `admin`, `ID=`, `redirect=`, `next=`, `/admin`, `/login`, `/config`, API keys, SSH keys, `.env` files, and anything else that looked interesting.

Two days in, I stopped and asked myself how a NASA scientist would share internal data. The answer was probably FTP, so I grepped for that.

One result looked like this:

`airs****.company1.nasa.gov/user/data/ftp/`

## The page

Opening it gave me a warning saying the content was reserved for NASA employees and that unauthorized access was punishable under federal law.

That wasn't the end of it. The Wayback Machine had cached a lot of links under the same path, like `airs***.company1.nasa.gov/user/data/ftp/project.hdf5`. They pointed to:

- ongoing research projects
- 3D models of satellites and rovers
- geospatial datasets
- scientific reports
- internal project documentation
- source code of internal tools

None of it should have been public.

## The report

I put together a detailed report and sent it to NASA's bug bounty team. They rated it P3 and sent me the LOR.

## What I took from it

Recon matters more than any single trick. The Wayback Machine is underrated. Think about how the people who built the thing actually work. And write down everything you do, because you'll need it for the report.

Thanks for reading, and happy hacking.
