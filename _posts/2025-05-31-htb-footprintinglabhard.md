---
title: HTB - Footprinting Lab (Hard)
categories: [htb]
tags: [htb]     # TAG names should always be lowercase
---

Mainly notes for self...



First we can start off with a routine NMAP scan:

```bash
nmap -sV 10.129.100.246
```

Output:
```bash
$ nmap -sV 10.129.100.246
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-05-30 19:51 CDT
Nmap scan report for 10.129.100.246
Host is up (0.21s latency).
Not shown: 995 closed tcp ports (reset)
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
110/tcp open  pop3     Dovecot pop3d
143/tcp open  imap     Dovecot imapd (Ubuntu)
993/tcp open  ssl/imap Dovecot imapd (Ubuntu)
995/tcp open  ssl/pop3 Dovecot pop3d
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.13 seconds
```

This shows us that the target is using Linux and has 5 ports open.
