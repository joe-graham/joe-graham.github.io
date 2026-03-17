---
layout: post
title:  "Hack The Box Write-up: Networked"
date:   2019-08-30 00:00:00 -0400
categories: posts htb pentesting
---
Networked is a Linux box created by Guly that is rated fairly easily by the HTB community. Even though it was fairly easy, I got some good practice with command injection vulnerabilities and circumventing file verification methods.

# Enumeration
Let’s start by taking a look at what services are running on the box using nmap.
```sh
Starting Nmap 7.80 ( https://nmap.org ) at 2019-08-30 16:16 EDT
Nmap scan report for 10.10.10.146
Host is up (0.26s latency).
Not shown: 65532 filtered ports
PORT    STATE  SERVICE VERSION
22/tcp  open   ssh     OpenSSH 7.4 (protocol 2.0)
80/tcp  open   http    Apache httpd 2.4.6 ((CentOS) PHP/5.4.16)
443/tcp closed https

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 1875.57 seconds
```

# HTTP
The only usable listening service at our disposal is HTTP. Browsing to the index page of the site reveals this page:

<!-- posts/networked/index_page.pnmg -->

Since there isn’t really anything useful on this page, lets throw dirb at the site to see if we can find any other pages.

```sh
root@kali:networked# dirb http://10.10.10.146 /usr/share/wordlists/dirb/big.txt
-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Fri Aug 30 17:14:07 2019
URL_BASE: http://10.10.10.146/
WORDLIST_FILES: /usr/share/wordlists/dirb/big.txt

-----------------

GENERATED WORDS: 20458                                                         

---- Scanning URL: http://10.10.10.146/ ----
==> DIRECTORY: http://10.10.10.146/backup/                                              
+ http://10.10.10.146/cgi-bin/ (CODE:403|SIZE:210)

==> DIRECTORY: http://10.10.10.146/uploads/                                                                                                                                                                                                                 
---- Entering directory: http://10.10.10.146/backup/ ----
(WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode -w if you want to scan it anyway)                                                                                                                                                                                                                  
---- Entering directory: http://10.10.10.146/uploads/ ----
                                                                                                                                                                                                                 
-----------------
END_TIME: Fri Aug 30 17:52:43 2019
DOWNLOADED: 40916 - FOUND: 1
```