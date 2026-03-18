---
layout: post
title:  "Hack The Box Write-up: iClean"
date:   2024-04-15 00:00:00 -0400
categories: posts htb pentesting
---
iClean is a Linux box created by LazyTitan33 that is rated medium by the HTB community. iClean’s exploit process involves exploiting a web application vulnerable to blind cross-site scripting (XSS), server-side template injection (SSTI), and then privilege escalation via an arbitrary write as root vulnerability in the sudo configuration.

## Enumeration

As usual, we start by enumerating the open ports and services on the system.

```sh
# Nmap 7.94SVN scan initiated Mon Apr 15 12:07:52 2024 as: nmap -p- -sV -oA 10.10.11.12 -v 10.10.11.12
Nmap scan report for capiclean.htb (10.10.11.12)
Host is up (0.027s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Apr 15 12:07:59 2024 -- 1 IP address (1 host up) scanned in 7.04 seconds
```

## HTTP

The only usable listening service at our disposal is HTTP. Browsing to the index page of the site reveals this page:

![Index page for a website called "Capiclean". Includes a Login button, a search bar, a hamburger menu, and the content "KEEP YOUR HOUSE CLEAN" "Let us take care of the choring so that you can focus on what really matters - running your business or enjoying your home."](/assets/iclean/index_page.png)

