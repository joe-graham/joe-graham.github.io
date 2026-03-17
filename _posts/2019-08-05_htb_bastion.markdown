---
layout: post
title:  "Hack The Box Write-up: Bastion"
date:   2019-08-05 00:00:00 -0400
categories: posts htb pentesting
---
This is my first in a series of write-ups on systems I’ve successfully exploited on HackTheBox. Bastion is a Windows host that at the time of writing has been rated fairly easy by other hackers, which was my experience as well. However, this system was still a fun system to exploit with a novel way of getting user access.
# Enumeration
```sh
root@kali:bastion# nmap -p- -sV 10.10.10.134
Starting Nmap 7.70 ( https://nmap.org ) at 2019-08-05 20:49 EDT
Nmap scan report for 10.10.10.134
Host is up (0.060s latency).
Not shown: 65522 closed ports
PORT      STATE SERVICE      VERSION
22/tcp    open  ssh          OpenSSH for_Windows_7.9 (protocol 2.0)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
49664/tcp open  msrpc        Microsoft Windows RPC
49665/tcp open  msrpc        Microsoft Windows RPC
49666/tcp open  msrpc        Microsoft Windows RPC
49667/tcp open  msrpc        Microsoft Windows RPC
49668/tcp open  msrpc        Microsoft Windows RPC
49669/tcp open  msrpc        Microsoft Windows RPC
49670/tcp open  msrpc        Microsoft Windows RPC
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 100.64 seconds
```
