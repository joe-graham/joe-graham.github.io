+++ 
draft = false
date = 2022-01-21T19:23:53-04:00
title = "Porting an Exploit for CVE-2016-6366"
description = "Several years ago I ported an exploit for a vulnerable firmware for the Cisco ASA that did not have a PoC written yet."
slug = "" 
tags = []
categories = []
externalLink = ""
series = []
+++
<script src="/lightbox/lightbox-plus-jquery.js"></script>

A few years ago I wrote a [blog post](https://ritcsec.wordpress.com/2017/05/17/improving-an-exploit-for-cve-2016-6366/) for a class I was taking in school exploring the Cisco Adaptive Security Appliance and trying to port the exploit to a known vulnerable version of the firmware that did not yet have an exploit written for it. As you can see from how the post ends, I ran out of time that semester and was unable to finish the exploit. In 2018 I picked the project back up and successfully ported it. In continuing my habit of picking up old projects years later, this post serves as an update to the original post explaining the problem and how I solved it.

## Picking Up Where We Left Off

The vulnerability in question is a buffer overflow in the Simple Network Management Protocol (SNMP) within the ASA, specifically the code implementing the getBulkRequest method. The overflow exists in the part of the code that reads in Management Information Base number sent as part of the request.

<center>
<a href="/posts/asa_cve_2016_6366/wireshark_screenshot.png" data-lightbox="wireshark_screenshot" data-title="The SNMP request sent when running the exploit.">
<img src="/posts/asa_cve_2016_6366/wireshark_screenshot_thumb.png" /></a>
</center>

The exploit patches the password verification function used by the ASA to check passwords. When throwing the [RiskSense improved exploit](https://github.com/RiskSense-Ops/CVE-2016-6366) at the ASA, instead of patching the function, the ASA crashes. The crashdump generated by the ASA looks like this:

<center>
<a href="/posts/asa_cve_2016_6366/crashdump.png" data-lightbox="crashdump" data-title="The crashdump generated by the ASA.">
<img src="/posts/asa_cve_2016_6366/crashdump_thumb.png" /></a>
</center>

## Debugging Shellcode with gdbserver

The shellcode used by the RiskSense exploit code is helpfully included in nasm format in the repo [here](https://github.com/RiskSense-Ops/CVE-2016-6366/blob/master/shellcode/shellcode.nasm). The shellcode is fairly straightforward: it stores the original values of the various registers on the stack, calls the sys_mprotect system call to allow the password check function to be modified, copies the patch code, then returns. The small amount of code means there were not that many areas where something could've went wrong. To investigate further, I used the gdbserver included with the ASA for debugging purposes to troubleshoot the shellcode. The asatools scripts discussed in the first blog post automates the process for patching the firmware to enable the gdbserver on startup.

Let's take another look at the crashdump. The instruction pointer is pointing at 0xcb980943, which is in the area of memory the stack is stored in. The shellcode makes changes to eax, esi, edi, ebx, and ebp. ebx, ebp, and edi are all set to the values that the shellcode changes them to before returning execution, which suggests that the code crashed towards the end of execution.

Using the gdbserver and a gdb client on a workstation connected to the ASA over a serial connection, we can set a breakpoint at the SNMP function right before it jumps to the shellcode and look at the values of the registers at that point as well as when the code crashes to investigate further.

## Troubleshooting the Shellcode

<center>
<a href="/posts/asa_cve_2016_6366/breakpoint1.png" data-lightbox="breakpoint1" data-title="Register values just before jumping to shellcode.">
<img src="/posts/asa_cve_2016_6366/breakpoint1_thumb.png" /></a>
<a href="/posts/asa_cve_2016_6366/breakpoint2.png" data-lightbox="breakpoint2" data-title="Register values just before returning execution from the shellcode.">
<img src="/posts/asa_cve_2016_6366/breakpoint2_thumb.png" /></a>
</center>

In a quirk of the shellcode that was either to make it more difficult for script kiddies to run the exploit on unpatched ASAs or an artifact from development that was accidentally left in, the shellcode includes a breakpoint just before returning execution, which made it much easier to troubleshoot further but was the actual cause for the crash - the hardware interrupt is not caught by a debugger, so the lina binary and thus the ASA crashes. However, even if you remove the breakpoint, the exploit would still break. As you can see from the register values, the stack pointers ebp and esp are off by four when reset by the shellcode. Because the stack is not being properly returned to its original structure prior to the exploit, whenever the ASA references something on the stack after exploitation, the value of whatever variable is going to be wrong, causing more problems.

## Solution

To fix the shellcode, I removed the hardware interrupt and added one more instruction that adds 4 bytes to ebp before the ebp offset is added back to ebp. You can see these changes in my fork of the RiskSense repo [here](https://github.com/joe-graham/CVE-2016-6366/commit/f8e7dbd23be4f690c8d071739d308aae1241defd).