---
title: Ticket8 - Linux workstation cannot mount the SMB share
description: "Looking back at LUM-1 through 7, every single one stayed inside Windows. I went back through the Canadian helpdesk postings I have been reading and cross-platform environment shows up in almost every one of them, while my portfolio had no Linux in it at all."
pubDate: 2026-09-23
heroImage: "https://ik.imagekit.io/stephanie/Ticket-8/LUM8-34.png?updatedAt=1790291962468"
badge: "IT Support"
tags:

  - Linux
  - SMB
  - File Services
---


# LUM-8 Linux

**Ticket:** LUM-8 </br>
**Date:** 2026-09-23 </br>
**Category:** File Services / Linux</br>
 **Environment:** LUMEN.LOCAL / DC01 (Windows Server 2022, 192.168.136.10) / LINUX01 (Ubuntu Desktop 24.04 LTS, 192.168.136.22) </br> 
 **Root cause:** LINUX01 had never been provisioned for domain file share access. Without `cifs-utils` the kernel could not mount an SMB share at all, and there was no credential storage and no persistent mount configured either.</br>

## The situation

Eighth ticket. And the first one that leaves Windows behind.

Looking back at LUM-1 through LUM-7, every single one stayed inside Windows. AD, permissions, GPO, DNS. LUM-7 took a step out into networking, but that was still a Windows client.

So I went back through Canadian helpdesk postings again. "Windows, macOS, and Linux" or "cross-platform environment" shows up in almost every one of them. My portfolio had not a single line of Linux in it. If an interviewer asked me "so you've only worked in Windows?", I'd have nothing to say.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-17.png?updatedAt=1790291962802)

So this time I decided to grow the lab itself. Stand up an Ubuntu workstation, then build a ticket around reaching the company file share from it. I already have a pile of ticket ideas waiting and here I am digging another hole. Of course I am.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-01.png?updatedAt=1790291961915) Here's the scenario. John Smith in IT gets a new Ubuntu machine. The CompanyData share opens fine in the file manager, but `/mnt/companydata` in the terminal is empty.

Scripts can't find anything. On his Windows machine the same share is mapped as a drive letter and it's still there after a reboot.

I set the priority to Medium. He can still see the files through the file manager, so he isn't fully blocked.

There's no fault injection in this ticket. **A freshly installed Ubuntu simply cannot mount an SMB share.** That state is the ticket. In the real world this is a provisioning gap on a new machine, and it comes in constantly whenever someone changes departments or gets new hardware.

## Building the lab was the first job

Every ticket so far ran inside a lab that already existed. This time I had to build a machine first. ![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-11.png?updatedAt=1790291961909)

<table><thead><tr><th>Item</th><th>Value</th></tr></thead><tbody><tr><td>OS</td><td>Ubuntu Desktop 24.04 LTS (64-bit)</td></tr><tr><td>vCPU</td><td>2</td></tr><tr><td>Memory</td><td>4 GB</td></tr><tr><td>Disk</td><td>25 GB (Split, not pre-allocated)</td></tr><tr><td>Network</td><td>NAT (same as DC01 and CLIENT01)</td></tr><tr><td>Hostname</td><td>linux01</td></tr></tbody></table>

I picked Desktop over Server. The ticket only works from a user's point of view if there's a scene where someone types `smb://` into a file manager and it doesn't work. That choice ended up being the most interesting part of the whole ticket. More on that later.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-16.png?updatedAt=1790291962421)

I did not join it to the domain. This is where I got confused for a while. Joining Linux to AD means setting up realmd and SSSD, which is Tier 3 territory, and more to the point **you don't need a domain join to use a file share.**

Which means LINUX01 ends up with two accounts. One is the local Ubuntu account `jsmith`, the other is `jsmith@lumen.local` in AD. Same name, completely different accounts. They live in different places and they have different passwords.

<table><thead><tr><th></th><th>Local jsmith</th><th>AD jsmith</th></tr></thead><tbody><tr><td>Stored in</td><td>/etc/shadow on LINUX01</td><td>Active Directory on DC01</td></tr><tr><td>Used for</td><td>Logging in, sudo</td><td>Connecting to the share</td></tr><tr><td>Valid on</td><td>This one machine</td><td>The whole domain</td></tr></tbody></table>

I lost real time to this during the lab. I'm not joking, it kept asking me for a password and I had no idea whether it wanted the AD one or the Linux one. My eyes were rolling. It asks every single time. EVERY TIME.

 > So I boiled it down to one rule. **If `sudo` is in the command, it's the local password. If you're connecting to the share, it's AD.** Anything with `sudo` in front of it is Linux local, no exceptions.

### Two things blocked me during install

The first was the network. On the `Connect to the internet` screen in the installer, no wired connection showed up. DC01 was definitely running. It only cleared up once I checked the adapter connection on the VMware side. ![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-23.png?updatedAt=1790291961938) So: power on DC01 first, every time. Seriously. Nothing works until the server is up, Linux or Windows.

The second was VMware Tools.

```
VMware Tools is no longer shipped with VMware Workstation for legacy guest
operating systems.
```

At first I thought something had gone wrong. It turns out Broadcom dropped the bundled per-guest Tools ISOs after acquiring VMware, and **Ubuntu 24.04 doesn't need them anyway.** `open-vm-tools` replaces them and ships with the distro. The legacy ISO in that link is for much older systems like CentOS 6.

I checked whether it was installed.

```
systemctl status open-vm-tools
```

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-19.png?updatedAt=1790291962549)

```
Unit open-vm-tools.service could not be found.
```

Not there. Probably because the network dropped during install. So I went to install it, and this time I was the problem. Of course I was. Rule of thumb: when the computer isn't listening to you, check your typing before you check anything else.

```
sodo apt install -y open-vm-tools open-vm-tooks-desktop
```

I typed `sodo` instead of `sudo`, and `tooks` instead of `tools`. And Linux answered like this.

```
Command 'sodo' not found, did you mean:
  command 'sudo' from deb sudo (1.9.15p5-3ubuntu5.24.04.2)
```

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-20.png?updatedAt=1790291962684) It catches the typo and tells you which package the command lives in. Compare that to Windows saying `is not recognized as an internal or external command` and the difference in useful information is obvious. You get the same screen when the command genuinely doesn't exist, and at that point you just run `sudo apt install <package>` and move on. Learned that here.

----------

## Capturing the state before touching anything

This is a habit I picked up in LUM-6. Record the state before you change it. You do not want to be in a position where you can't tell what was already there and what you broke.

I started with the IP configuration on LINUX01.

```
ip addr show
```

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-21.png?updatedAt=1790291962489)

<table><thead><tr><th>Item</th><th>Value</th></tr></thead><tbody><tr><td>Interface</td><td>ens33</td></tr><tr><td>State</td><td>state UP</td></tr><tr><td>IPv4</td><td>192.168.136.22/24</td></tr><tr><td>Assignment</td><td>dynamic (DHCP)</td></tr><tr><td>MAC</td><td>00:0c:29:e6:59:91</td></tr></tbody></table>

The word `dynamic` is the important one. It means the address came from DHCP, and the DHCP server on this network is DC01. So that single line proves LINUX01 is already talking to DC01.

`00:0c:29` is the MAC prefix VMware uses. That's the kind of detail you actually use in the field to guess what a device is.

DNS next.

```
resolvectl status
```

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-22.png?updatedAt=1790291962642)

```
Current DNS Server: 192.168.136.10
DNS Servers: 192.168.136.10
DNS Domain: lumen.local
Protocols: +DefaultRoute -LLMNR -mDNS
```

DNS points at DC01 and the search domain is `lumen.local`. That confirms up front that this ticket won't overlap with LUM-7's root cause.

The `-mDNS` also caught my eye. It means multicast DNS is off, and honestly I got lucky there. Ubuntu's systemd-resolved sends anything ending in `.local` to multicast DNS by default. Our domain is `LUMEN.LOCAL`. If mDNS had been enabled, `dc01.lumen.local` would have leaked out as a broadcast instead of going to the DNS server, and resolution would have failed.

This is exactly why Microsoft tells you not to use `.local` for an AD domain. It didn't bite me this time, but it's a trap you can hit at any point, so I'm writing it down.

I checked the DC01 side too.

```
Get-SmbShare -Name CompanyData
(Get-Acl "C:\CompanyData\IT").Access
```

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-24.png?updatedAt=1790291962063)

`LUMEN\IT-Users` has Modify. jsmith is a member of IT-Users, so permissions are fine. I confirmed it with Effective Access as well.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-25.png?updatedAt=1790291962148)

Read, write and folder creation are all allowed. Only Full control and Take ownership are denied. Exactly as intended. That's **evidence that permissions were never the problem**, captured before I touched anything.

----------

## Reproducing the symptom

I started as the user would. File manager, `Other Locations`, then `smb://dc01.lumen.local/CompanyData`.

An authentication dialog came up. `LUMEN` for Domain, `jsmith` for Username, and the AD password.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-27.png?updatedAt=1790291962824)

Failed. Tried again. Failed again.

Hmm. That's odd. Over to the terminal.

```
sudo mkdir -p /mnt/companydata
sudo mount -t cifs //dc01.lumen.local/CompanyData /mnt/companydata -o username=jsmith
```

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-28.png?updatedAt=1790291962600)

```
mount: /mnt/companydata: mount(2) system call failed: No route to host.
```

And this is where the real ticket starts. Why on earth is it telling me the network is unreachable?

## The error message lied to me

`No route to host`. There's no path. The network can't be reached.

Except this machine has internet. I just pulled packages down with `apt install`. It got its address from DC01 over DHCP. None of this added up.

I could have taken the message at face value and started digging through routing tables and firewall rules. Instead I applied the same sequence I used in LUM-7. **Work up from the bottom of the stack and test whether the message is actually telling the truth.**

### 1. Is there connectivity?

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-32-1.png?updatedAt=1790355871106)

```
ping -c 4 192.168.136.10
```

```
4 packets transmitted, 4 received, 0% packet loss
```

Linux `ping` runs forever by default, so you need `-c 4` to cap it. Windows stops after four on its own, which is the opposite habit.

Zero loss. **Not the network.**

### 2. Does the name resolve?

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-32-2.png?updatedAt=1790355871118)

```
getent hosts dc01.lumen.local
```

```
192.168.136.10  dc01.lumen.local
```

Using `getent` instead of `dig` here was deliberate.

<table><thead><tr><th></th><th>dig</th><th>getent</th></tr></thead><tbody><tr><td>Queries</td><td>The DNS server directly</td><td>The whole system name resolution chain</td></tr><tr><td>Path taken</td><td>DNS only</td><td>/etc/hosts → mDNS → DNS</td></tr><tr><td>Shows you</td><td>What the DNS server answers</td><td>What an application actually gets back</td></tr></tbody></table>

If you want to see what the `mount` command itself experiences, `getent` is the right tool. On the Windows side this is the same shape as `nslookup` working while the actual connection fails. That's the gap I ran into in LUM-7.

**Not DNS.**

### 3. Is the port open?

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-32-2.png?updatedAt=1790355871118)

```
nc -zv dc01.lumen.local 445
```

```
Connection to dc01.lumen.local (192.168.136.10) 445 port [tcp/microsoft-ds] succeeded!
```

`nc` is netcat. `-z` checks whether the port is open without sending any data, and `-v` prints the result in a readable form. It's the counterpart to `Test-NetConnection -Port 445` on Windows.

445 is open. **Not the firewall.** So what is it? The network clearly isn't the problem.

### Summary

<table><thead><tr><th>What the message pointed at</th><th>What I found</th></tr></thead><tbody><tr><td>Routing</td><td>ping, 0% loss</td></tr><tr><td>Name resolution</td><td>getent resolved fine</td></tr><tr><td>Port reachability</td><td>445 open</td></tr></tbody></table>

**Every layer the message implicated was healthy.** Which means the message was wrong.

## There were actually two causes

### One: the mount helper wasn't there

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-48.png)

```
which mount.cifs
dpkg -l | grep cifs-utils
```

Look closely and you can see ping working three lines up, while `which mount.cifs` returns nothing at all. In hindsight, empty output is the thing to watch for in Linux.

`mount -t cifs` isn't handled by the kernel on its own. It calls a helper program at `/sbin/mount.cifs`, and the package that provides it, `cifs-utils`, wasn't installed. It isn't part of a default Ubuntu Desktop install.

That's the root cause. More broadly, **this workstation had never been set up to use a domain file share.** No mount helper, no credential storage, no persistent mount. None of it.

### Two: I got the password wrong

I tried authenticating from the file manager one more time. This time it worked. The two earlier attempts had a typo in the password.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-31.png?updatedAt=1790291962458)

On a whim I retyped the AD password in Linux one more time, and the share just opened.

So `No route to host` was **the message returned for an authentication failure.** The CIFS kernel module hands back an error code called `EHOSTUNREACH` in situations that have nothing to do with routing, and `mount` prints that verbatim as "No route to host."

This is the same family of trap as misreading `bad option` as a syntax error back in LUM-7. **An error message is a clue, not a conclusion.** That time the real information was on the second line. This time the message was pointing at the wrong layer entirely.

## So why did the file manager work?

This is the most interesting part of the ticket.

Once the password was right, Finance, HR, IT and Sales showed up in the file manager. With `cifs-utils` still not installed.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-33.png?updatedAt=1790291962363)

I checked from the terminal.

```
ls /mnt/companydata
```

Empty. Same share, visible from one side only.

The file manager uses something different called **gvfs**. It runs inside the user's desktop session rather than in the kernel, so it doesn't need `cifs-utils`. It attaches under `/run/user/1000/gvfs/` instead, which isn't a standard filesystem path.

<table><thead><tr><th></th><th>gvfs (file manager)</th><th>cifs (kernel mount)</th></tr></thead><tbody><tr><td>Runs in</td><td>The user session</td><td>The kernel</td></tr><tr><td>Package needed</td><td>Ships by default</td><td>cifs-utils</td></tr><tr><td>Path</td><td>/run/user/1000/gvfs/...</td><td>/mnt/companydata</td></tr><tr><td>Visible in terminal</td><td>Effectively no</td><td>Yes</td></tr><tr><td>After logout</td><td>Gone</td><td>Still there</td></tr><tr><td>Usable from scripts</td><td>Painful</td><td>Yes</td></tr></tbody></table>

So from the user's side it looks like this. **The file manager shows everything, and the script can't find a thing.** The user sees the folders with their own eyes and assumes it's working, when it's really only half working.

This shows up constantly in real support work. And the fact that it looks fine in the file manager is itself what makes it hard to diagnose. The moment someone says "but it works for me," tracing the cause gets a lot harder.

## The fix

### Install the package

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-34.png?updatedAt=1790291962468)

```
sudo apt install -y cifs-utils
```

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-35.png?updatedAt=1790291962606) The command that gave me nothing earlier finally answers with `/usr/sbin/mount.cifs`. Finally.

### The credentials file

This is the security point of the ticket.

If you put `password=` straight into the command line, it ends up in your shell history and in the process list in plain text. If you put it in `/etc/fstab`, that file is readable by everyone. So you pull it out into its own file and lock the permissions down.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-36.png)

```
sudo nano /etc/samba/creds-companydata
```

The file is three lines.

```
username=jsmith
password=<AD password>
domain=LUMEN
```

And here's where I got stuck again. `/etc/samba` didn't exist. `nano` will create a file but it won't create a directory. Which makes sense once you think about it: I never installed the samba **server** package, so there's no reason that folder would be there. We're only running a client here.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-37.png?updatedAt=1790291962180)

```
sudo mkdir -p /etc/samba
```

Once the directory exists, go back to the nano window, hit `Ctrl` + `O` to save and `Ctrl` + `X` to get out.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-38.png?updatedAt=1790291962417) Then lock the permissions.

```
sudo chmod 600 /etc/samba/creds-companydata
ls -l /etc/samba/creds-companydata
```

```
-rw------- 1 root root 52 Sep 23 16:22
```

`600` means the owner can read and write and nobody else can do anything. The leading 6 is the owner (read 4 plus write 2), and the two zeros are group and everyone else.

The size showing as 52 bytes is worth checking too. If it says 0, nothing got saved.

There's no equivalent step on Windows. Credential Manager handles it for you. On Linux you do it yourself, and being able to explain that difference is what makes cross-platform experience real rather than claimed.

### Checking the UID

```
id jsmith
```

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-40-1.png)

```
uid=1000(jsmith) gid=1000(jsmith) groups=1000(jsmith),4(adm),24(cdrom),27(sudo),...
```

Why does this number matter? **SMB has no concept of Unix permissions.** Mount it as-is and the whole share comes up owned by root, so a normal user can read but not write.

DC01 sees "LUMEN\jsmith connected." But from LINUX01's side there's no way to know which local user that AD account corresponds to. There's no domain join, so there's no link between them. That's why you have to wire it up by hand with `uid=1000, gid=1000`.

Skip this and you get a second ticket immediately: "it's mounted but I can't save anything."

### The persistent mount

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-39.png?updatedAt=1790291962513)

```
sudo nano /etc/fstab
```

Added at the bottom, as a single line.

```
//dc01.lumen.local/CompanyData /mnt/companydata cifs credentials=/etc/samba/creds-companydata,uid=1000,gid=1000,file_mode=0664,dir_mode=0775,vers=3.0,_netdev,nofail 0 0
```

Option by option:

<table><thead><tr><th>Option</th><th>What it does</th></tr></thead><tbody><tr><td>credentials=</td><td>Path to the credentials file, so the password never goes in fstab itself</td></tr><tr><td>uid=1000, gid=1000</td><td>Maps mounted files to the local user instead of root</td></tr><tr><td>file_mode=0664</td><td>File permissions</td></tr><tr><td>dir_mode=0775</td><td>Directory permissions. Directories need execute to enter, so they're one higher than files</td></tr><tr><td>vers=3.0</td><td>Pins the SMB protocol version and documents that SMB1 is not in use</td></tr><tr><td>_netdev</td><td>Waits until networking is up before mounting</td></tr><tr><td>nofail</td><td>Lets the machine finish booting even if the mount fails</td></tr></tbody></table>

`nofail` matters more than it looks. Leave it out and LINUX01 will hang partway through boot whenever DC01 is off. That one causes real incidents.

```
sudo mount -a
```

One notice came back.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-40-2.png)

```
mount: (hint) your fstab has been modified, but systemd still uses
the old version; use 'systemctl daemon-reload' to reload.
```

Not an error, a hint. The mount already went through, and it's telling me to let systemd know fstab changed. So I did what it said.

```
sudo systemctl daemon-reload
```

## Verification

### Client side

```
findmnt /mnt/companydata
ls -la /mnt/companydata
```

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-42.png?updatedAt=1790291962490)

```
TARGET            SOURCE                          FSTYPE OPTIONS
/mnt/companydata  //dc01.lumen.local/CompanyData  cifs   rw,relatime,vers=3.0,...
```

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-40-4.png)

```
drwxrwxr-x? 2 jsmith jsmith 0 Aug 23 02:19 Finance
drwxrwxr-x? 2 jsmith jsmith 0 Aug 14 00:20 HR
drwxrwxr-x  2 jsmith jsmith 0 Aug 14 00:20 IT
drwxrwxr-x? 2 jsmith jsmith 0 Aug 14 00:21 Sales
```

`FSTYPE cifs`. That's a kernel mount, not gvfs. `rw` means read and write, and `vers=3.0` confirms the version I pinned is what got negotiated.

The owner reads `jsmith jsmith`, not root. The uid/gid options took effect.

The `?` after the folder permissions means the ACL information couldn't be read. SMB has no concept of Linux ACLs, so that's normal. IT is the only folder without one, because it's the only one with an explicit ACL on it. That's the `LUMEN\IT-Users` Modify permission I confirmed earlier. **The screenshot I took before starting is what answered that question.**

I ran a write test too.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-41.png?updatedAt=1790291962612)

```
touch /mnt/companydata/IT/lum8-write-test.txt
```

That left the reboot test. Get fstab wrong and the machine hangs on boot, so this one isn't optional.

```
sudo reboot
```

Checked again once it came back up.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-43.png?updatedAt=1790291962794)

Mounted automatically.

### Server side

Checking only from the client gets you halfway. I looked from DC01 as well.

```
Get-SmbSession | Format-Table ClientComputerName, ClientUserName, Dialect, NumOpens -AutoSize
Get-ChildItem C:\CompanyData\IT
```

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-44.png?updatedAt=1790291961883)

```
ClientComputerName  ClientUserName  Dialect  NumOpens
192.168.136.22      LUMEN\jsmith    3.0      0
```

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-45.png?updatedAt=1790291962035)

```
Mode    LastWriteTime        Length Name
-a----  8/13/2026  9:22 PM       23 IT-Test.txt
-a----  9/23/2026  1:33 PM        0 lum8-write-test.txt
```

A file I created with a single `touch` on LINUX01 is sitting on a Windows server's disk. That screen genuinely surprised me. I knew it in my head, but seeing it is different.

`Dialect 3.0` confirms that the `vers=3.0` in fstab is what actually got negotiated. That means I can prove SMB1 isn't in use using data from the server side, which is the kind of thing a security audit asks about.

And here's the observation that matters most. **LINUX01 is not domain joined, and yet DC01 identifies this session as `LUMEN\jsmith`.**

SMB authentication and domain join are separate things. A domain join is about registering a computer account, applying GPO and getting single sign-on. SMB authentication is about handing over credentials each time you connect. Windows has the same capability. It's exactly what the "Connect using different credentials" checkbox does when you map a network drive. On Linux there's no domain join, so it always works this way.

I checked the event log too, filtering the Security log for event ID 4624.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-8/LUM8-47.png?updatedAt=1790291962249)

4624 is a successful logon, and Logon Type 3 within it means a network logon. File share access is always Type 3. Where `Get-SmbSession` shows you the current state, the event log leaves a record of the past. Put the two together and you can put "when did authentication actually happen" into the ticket.

## What I took away

**Take an error message literally and you'll dig in the wrong place.**

`No route to host` said the network was the problem. But ping worked, the name resolved, and 445 was open. Every piece of evidence contradicted the message, and the real cause was a mistyped password. I came close to repeating the exact mistake I made in LUM-7 when I read `bad option` as a syntax error.

Going layer by layer to **disprove the message** is diagnosis too. Confirming "not this" three times wasn't wasted effort, it was how the cause got narrowed down.

**The state that looks like it's working is the hardest one.**

The folders were right there in the file manager. From the user's side, it works. And the script still fails. Without knowing that gvfs and a kernel mount operate at different layers, this ticket can't be explained at all. Between "it works" and "it doesn't" there's a third state: "it half works."

**The safety nets Windows gives you don't exist on Linux.**

In LUM-7, DNS was broken and LLMNR and NetBIOS still answered short-name lookups. Back then that got in the way of diagnosis. This time it's the reverse. Linux has no fallback path like that. Credential storage, account mapping, all of it is manual. I only realised how much Windows had been doing on my behalf after building each piece by hand on Linux.

**Domain join and authentication are two different conversations.**

Confirming that with server-side data is the biggest thing I got out of this ticket. A machine that isn't in the domain authenticates as an AD account and writes files. And in the real world this setup is far more common. Not many companies stand up and maintain realmd and SSSD for a handful of Linux boxes.

## What's left

Storing credentials in a static file has a limit. **If the password changes, the mount fails silently at the next boot.** No error, and nobody knows until a user reports it.

A Kerberos-based mount (`sec=krb5`) would remove the stored password entirely, but it requires a domain join. Splitting that out into its own ticket.

The gvfs mount is also still alive alongside the fstab mount. The same share is reachable through two different paths, so users need documentation telling them which one to use. Logging that as a follow-up observation as well.