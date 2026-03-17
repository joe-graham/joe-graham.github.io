---
layout: post
title:  "Hack The Box Write-up: Bastion"
date:   2019-08-05 00:00:00 -0400
categories: posts htb pentesting
---
This is my first in a series of write-ups on systems I’ve successfully exploited on HackTheBox. Bastion is a Windows host that at the time of writing has been rated fairly easy by other hackers, which was my experience as well. However, this system was still a fun system to exploit with a novel way of getting user access.

## Enumeration

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

One of the first things that stands out to me is that this host has OpenSSH listening on port 22. [Microsoft](https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_overview) recently integrated OpenSSH into Windows, but this is the first time I’ve seen it running as a service on a Windows host in the wild.
We can’t do anything with SSH right now, but this does indicate that this host is a Windows 10 or Windows Server 2016 or 2019 box, since OpenSSH is only available on this flavor of Windows. We will come back to this service later on in the write-up.

## SMB

Other than SSH and WinRM, the only other listening useful service is SMB. For SMB, I usually use a mix of enum4linux and smbmap to try to enumerate information about listening shares and the level of access provided by the server.
First, I tried enumerating access using an anonymous logon, which would be rare on such a modern version of Windows, but is still worth a shot.

```sh
root@kali:smb# enum4linux 10.10.10.134
Starting enum4linux v0.8.9 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Mon Aug  5 20:58:24 2019

--snip--
[E] Server doesn't allow session using username '', password ''. Aborting remainder of tests.
```

Unfortunately, anonymous access didn’t get us anything. It has, thankfully, gotten progressively more difficult to set up anonymous SMB shares in Windows over the years, so this is pretty realistic.
One other thing I like to try to enumerate access is by using the built-in Guest account with no password. Let’s give that a shot.

```sh
root@kali:smb# enum4linux -u Guest 10.10.10.134
Starting enum4linux v0.8.9 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Mon Aug  5 21:01:33 2019

--snip--
 ==================================================== 
|    Enumerating Workgroup/Domain on 10.10.10.134    |
 ==================================================== 
[E] Can't find workgroup/domain


 ===================================== 
|    Session Check on 10.10.10.134    |
 ===================================== 
Use of uninitialized value  in concatenation (.) or string at ./enum4linux.pl line 437.
[+] Server 10.10.10.134 allows sessions using username 'Guest', password ''
Use of uninitialized value  in concatenation (.) or string at ./enum4linux.pl line 451.
[+] Got domain/workgroup name: 

 =========================================== 
|    Getting domain SID for 10.10.10.134    |
 =========================================== 
Use of uninitialized value  in concatenation (.) or string at ./enum4linux.pl line 359.
Cannot connect to server.  Error was NT_STATUS_LOGON_FAILURE
[+] Can't determine if host is part of domain or part of a workgroup
enum4linux complete on Mon Aug  5 21:01:44 2019
```

It looks like the server allows connections as Guest! Running enum4linux with the `-a` parameter only listed what shares the Guest user is able to access, and nothing else helpful.

```sh
root@kali:smb# enum4linux -a -u Guest 10.10.10.134
Starting enum4linux v0.8.9 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Mon Aug  5 21:03:25 2019

--snip--

 ========================================= 
|    Share Enumeration on 10.10.10.134    |
 ========================================= 
Use of uninitialized value  in concatenation (.) or string at ./enum4linux.pl line 640.
do_connect: Connection to 10.10.10.134 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	Backups         Disk      
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
Reconnecting with SMB1 for workgroup listing.
Failed to connect with SMB1 -- no workgroup available

--snip--
```

I didn’t expect to be able to access the default administrative shares (ADMIN$, C$, and IPC$), and attempting to connect using smbclient confirmed that.
However, the Backups share seems interesting to us. Let’s see what’s inside that share.

```sh
root@kali:smb# smbclient -U Guest //10.10.10.134/Backups
Enter WORKGROUP\Guest's password: 
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Mon Aug  5 21:09:35 2019
  ..                                  D        0  Mon Aug  5 21:09:35 2019
  ASwYWuILip                          D        0  Mon Aug  5 20:51:59 2019
  note.txt                           AR      116  Tue Apr 16 06:10:09 2019
  oTWBZwpEdv                          D        0  Mon Aug  5 20:51:50 2019
  SDT65CB.tmp                         A        0  Fri Feb 22 07:43:08 2019
  tmp                                 A      116  Mon Aug  5 21:09:35 2019
  WindowsImageBackup                  D        0  Fri Feb 22 07:44:02 2019

		7735807 blocks of size 4096. 2788438 blocks available
```

