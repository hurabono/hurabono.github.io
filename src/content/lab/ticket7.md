---
title: Ticket7 - DNS name resolution failure
description: "Looking back at LUM-1 through 6, every single one stayed inside AD, permissions and GPO. I went back through the Canadian helpdesk postings I have been reading and almost every one of them has basic networking knowledge sitting in the requirements."
pubDate: 2026-09-09
heroImage: "https://ik.imagekit.io/stephanie/Ticket-7/LUM7-01.png?updatedAt=1789013073792"
badge: "IT Support"
tags:

  - Networking
  - DNS
---


## LUM-7 DNS name resolution failure

**Ticket:** LUM-7 </br>
**Date:**  2026-09-09</br>
**Category:** Networking / DNS</br>
**Environment:** LUMEN.LOCAL / DC01 (Windows Server 2025, 192.168.136.10) / CLIENT01 (Windows 11, 192.168.136.21)</br>
**Root cause:** The IPv4 DNS server on CLIENT01 was manually set to a public resolver (8.8.8.8) instead of the domain DNS server, so the client could not resolve internal FQDNs or AD SRV records</br>

----------

## The situation

Seventh ticket. And the first one that steps outside AD.

Looking back at LUM-1 through 6, every single one stayed inside AD, permissions and GPO. I went back through the Canadian helpdesk postings I have been reading and almost every one of them has "basic networking knowledge" sitting in the requirements. My portfolio had not a single line of it. So this time I deliberately went down to the network layer.

Here is the scenario. Emily Brown from IT files a ticket. She is following an internal document that links to `\\DC01.lumen.local\CompanyData` and it will not open. But her usual mapped drive is perfectly fine and the internet is normal. A colleague opens the same link without any trouble.

I set the priority to Medium. The mapped drive is alive, so she is not fully blocked. I was going to go with High and then dropped it after actually running the lab. Whether that call was right is something I come back to later. Short version: the state of that machine was far more dangerous than it looked.

The fault injection is simple. Change the DNS server address on CLIENT01 to 8.8.8.8.

----------

## Blocked right out of the gate

I went to open Control Panel like the guide said. Instead I got this.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-01.png?updatedAt=1789013073792)

"This operation has been cancelled due to restrictions in effect on this computer."

I thought the lab had broken. Then it occurred to me that this means a policy is in effect. So I checked.

```
gpresult /r /scope:user
```

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-02.png?updatedAt=1789013073189)

A GPO called `IT-LabUser-Restrictions` was being applied. ebrown sits in the IT OU and is a member of IT-Users, so it was doing exactly what it was supposed to do. Not broken. Working.

Control Panel being blocked does not mean the work is blocked. The `NoControlPanel` policy only closes off the Explorer UI, it has no effect on PowerShell cmdlets. So I just went to PowerShell.

First I recorded the state before changing anything. That habit came out of LUM-6. If you do not know the original value, you cannot put it back.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-04.png?updatedAt=1789013073507)

<table><thead><tr><th>Item</th><th>Value</th></tr></thead><tbody><tr><td>DNS Servers</td><td>192.168.136.10</td></tr><tr><td>IPv4 Address</td><td>192.168.136.21</td></tr><tr><td>Default Gateway</td><td>192.168.136.2</td></tr><tr><td>DHCP Server</td><td>192.168.136.10</td></tr><tr><td>Adapter name</td><td>Ethernet0</td></tr><tr><td>DHCP Enabled</td><td>Yes</td></tr></tbody></table>

Two things came out of this. One, the adapter is called `Ethernet0`, not `Ethernet`. That is VMware's default naming. Two, this machine is on DHCP. That second one matters later when I go to undo the change.

I also noticed `Node Type: Hybrid` and `NetBIOS over Tcpip: Enabled` and skimmed straight past them. I had no idea those two lines were about to become the whole point of this ticket.

----------

## Blocked a second time

Went to inject the fault and got blocked again, ha.
They say at school that red text in a console is always beautiful.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-06.png?updatedAt=1789013074090)

```
Set-DnsClientServerAddress : Access to a CIM resource was not available to the client.
CategoryInfo : PermissionDenied
```

I stared at it for a while assuming it was an account problem. It was not. I had not opened the window as administrator.

The title bar just said `Windows PowerShell`. When it is elevated it says `Administrator: Windows PowerShell`. And the `gpresult` output I had captured earlier had `Medium Mandatory Level` printed at the bottom of it. An elevated process shows `High Mandatory Level`. The evidence was already in my hand and I did not read it.

Opened it as administrator and injected the fault.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-07.png?updatedAt=1789013073997)

```
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 8.8.8.8
ipconfig /flushdns
```

Confirmed DNS had changed to `{8.8.8.8}`.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-08.png?updatedAt=1789013073271)

And give it a flush with flushdns. Down it goes.

----------

## Reproducing the symptoms

Three things to confirm.

**The internet is fine.**

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-09.png?updatedAt=1789013072731)

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-10.png?updatedAt=1789013073991)
Google search works, and `Test-NetConnection www.google.com -Port 443` comes back True.

**The FQDN share does not work.**

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-11.png?updatedAt=1789013074425)

```
Windows cannot access \\DC01.lumen.local\CompanyData
```

**Group Policy fails.**

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-12.png?updatedAt=1789013074180)

```
Computer policy could not be updated successfully.
a) Name Resolution failure on the current domain controller.

```
This part actually made me laugh. Windows had listed the cause as its own first candidate, correctly. Reading an error message all the way to the end turns out to give you the answer more often than you would think.

----------

## Finding the cause

### The IP works but the name does not

This is the fork in the road for the whole ticket.

```
ping 192.168.136.10        → success
ping DC01.lumen.local      → failure

```

The packets get there fine. It stops at the step where a name has to become an IP address. That means this is not a connectivity problem, it is a name resolution problem. That single line cut the search area right down.

### I asked the same question to two different servers

I queried the SRV record. That is the special record a domain joined PC uses to find a DC.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-19.png?updatedAt=1789013073988)

```
nslookup -type=SRV _ldap._tcp.dc._msdcs.lumen.local
Server:  dns.google
Address: 8.8.8.8
*** dns.google can't find _ldap._tcp.dc._msdcs.lumen.local: Non-existent domain

```

The part to look at here is not the result, it is the **top two lines**. `Server` is "who am I asking right now". That is Google sitting there, not the domain controller. At this point it was effectively over.

Just to be certain I put the same question directly to DC01.
Turns out if you ask a computer a straight question, it gives you a straight answer.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-20.png?updatedAt=1789013073722)

```
nslookup -type=SRV _ldap._tcp.dc._msdcs.lumen.local 192.168.136.10

_ldap._tcp.dc._msdcs.lumen.local  SRV service location:
    priority = 0
    weight = 100
    port = 389
    svr hostname = dc01.lumen.local

```

Success. nslookup lets you name the server you want to query as a second argument. **When the same question gets different answers, the fault is on the client, not the server.** That one technique immediately settled whether I should be looking at DC01 or at CLIENT01.

It also explains why the GPO failed. A domain joined PC does not find a DC from an A record alone. It queries this SRV record to decide which DC and which port (389) to connect to. When that fails, policy, logon scripts and drive mapping all go down together.

----------

## But then something strange happened

As I kept going, three checks came back healthy.

### 1. nltest succeeded

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-21.png?updatedAt=1789013073891)

```
nltest /dsgetdc:lumen.local
DC: \\DC01.lumen.local
Address: \\192.168.136.10
The command completed successfully
```

Huh? DNS is dead and it found the DC anyway. That threw me.


### 2. gpresult also looked healthy

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-23.png?updatedAt=1789013073468)

```
Group Policy was applied from: DC01.lumen.local
Last time Group Policy was applied: 11:58:12 PM
IT-LabUser-Restrictions

```

`gpupdate /force` had failed right above it, and `gpresult /r` was reporting normal. Having the two sitting on the same screen made it more confusing, not less.

I understood this later. **`gpresult` reads the cached result of the last successful run.** It shows you the past, not the present. `gpupdate` actually goes out and tries to pull policy right now. So one succeeds and one fails.

> `gpresult` talks about the past. `gpupdate` talks about the present.

### 3. The share opened under the short name

This was the one that really surprised me.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-26.png?updatedAt=1789013073995)

```
\\DC01\CompanyData
```

It just opened. Finance, HR, IT and Sales, all there. With DNS sitting dead on 8.8.8.8.

So `\\DC01.lumen.local\CompanyData` fails and `\\DC01\CompanyData` works. Same server, same folder...??

----------

## It came down to one dot

I checked with ping.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-27.png?updatedAt=1789013074345)

```
ping DC01
Pinging DC01.local [fe80::4133:e7d9:cd5b:1239%10] with 32 bytes of data:
Reply from fe80::4133:e7d9:cd5b:1239%10: time<1ms

ping DC01.lumen.local
Ping request could not find host DC01.lumen.local.

```

I looked at the successful side more closely. Two things were off.

-   It resolved to **`DC01.local`**, not `DC01.lumen.local`
-   The reply came from a **`fe80::`** link local address, not from `192.168.136.10`

Neither of those values is possible if the answer had come through domain DNS. It had taken a completely different route.

Reading up on it, Windows does not use DNS alone to resolve a name. The presence or absence of a dot splits it down two different paths.

<table><thead><tr><th>Name</th><th>DNS</th><th>mDNS / LLMNR</th><th>NetBIOS</th><th>Result</th></tr></thead><tbody><tr><td>DC01</td><td>fails</td><td>available</td><td>available</td><td>success</td></tr><tr><td>DC01.lumen.local</td><td>fails</td><td>not available</td><td>not available</td><td>failure</td></tr></tbody></table>

A single label name like `DC01` with no dot in it can still shout "is DC01 out there?" over multicast or broadcast even after DNS fails. DC01 is on the same subnet, so the shout reaches it and it answers directly.

`DC01.lumen.local`, on the other hand, is treated as a fully qualified domain name the moment the dot appears. LLMNR is single label only, and a NetBIOS name cannot contain a dot. There is no alternative route at all. DNS is the only one, and DNS was dead, so it died with it.

**The short name has two extra emergency exits. The FQDN has one front door.**

That is also why `nltest` succeeded earlier. Netlogon found the DC over broadcast. But the DNS based SRV lookup that actual policy processing depends on was still failing, so `gpupdate` failed.

----------

## Why this is scary

This is where setting the priority to Medium started to bother me.

What a user touches every day is the mapped drive. `net use` showed it mapped as `\\DC01\CompanyData`. No dot. So it keeps opening even with DNS dead. The user has no reason to notice anything is wrong.

Meanwhile, here is what breaks quietly.

-   Group Policy stops refreshing. Security settings, software deployment, drive mapping preferences, all of it
-   Logon scripts fail if they are FQDN based
-   Kerberos SPNs are FQDN based, so authentication falls back to NTLM. It still works, at a lower security level
-   Certificate autoenrollment and most software update mechanisms use FQDNs

You end up with **security policy that has not refreshed in weeks and nobody knows**. Because from the user's side nothing looks wrong.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-27.png?updatedAt=1789013074345)
And this workaround path runs over a `fe80::` link local address, which means it does not cross a router. Translated into a real environment, it looks like this.

> The people on the same floor can open the share and the people at another site cannot. Same DNS misconfiguration, different symptoms depending on who you ask.

Intermittent faults like that are the hardest ones to catch. Something that dies completely gets found quickly. Something half alive can waste days.

If Emily had shrugged and said "the mapped drive works so whatever", this machine would have stayed in that state. Filing the ticket instead of working around it was the right call.

----------

## The fix

I had to remember this machine is on DHCP. Hardcoding 192.168.136.10 gives you the correct value but not the original state. The original state was DHCP handing it out automatically.

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ResetServerAddresses
```

`-ResetServerAddresses` clears the manual setting and puts the adapter back to using whatever DHCP provides. **You are restoring the method, not the value.**

Except it failed the first time.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-28.png?updatedAt=1789013073431)
```
No MSFT_DNSClientServerAddress objects found with property 'InterfaceAlias' equal to 'Ethernet'
CategoryInfo : ObjectNotFound
```

I typed `Ethernet` instead of `Ethernet0`. Dropped the zero off the end.
Ugh, another mess caused entirely by me mistyping a name.

Red text showed up again and my reflex was to suspect permissions, but this time it was not that. `CategoryInfo` tells you which is which.

<table><thead><tr><th>CategoryInfo</th><th>Meaning</th><th>What to do</th></tr></thead><tbody><tr><td>PermissionDenied</td><td>Not enough privilege</td><td>Reopen the window as administrator</td></tr><tr><td>ObjectNotFound</td><td>The target name is wrong</td><td>Check the name and correct it</td></tr></tbody></table>

I hit both of them inside one ticket. When red text appears, read the category first.

Fixed the name, ran it again, and it came back to `{192.168.136.10}`.
![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-29.png?updatedAt=1789013074394)
![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-30.png?updatedAt=1789013074186)
```
ipconfig /flushdns
ipconfig /registerdns
```

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-31.png?updatedAt=1789013073012)
As a side note, I tried to restart Netlogon from `services.msc` and got blocked again. Every menu item was greyed out. Same cause, an unelevated window, but the CLI shouts at you in red while the GUI just quietly disables things. The GUI version was actually more confusing, because it says nothing at all.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-32.png?updatedAt=1789013074194)
One more confirmation shot, and then into verification.

----------

## Verification

I re-ran everything that had failed, in the same order.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-33.png?updatedAt=1789013074301)

```
nslookup DC01.lumen.local
Server:  UnKnown
Address: 192.168.136.10
Name:    DC01.lumen.local
Address: 192.168.136.10

```


`Server` had changed from 8.8.8.8 to 192.168.136.10.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-34.png?updatedAt=1789013074403)

`Server: UnKnown` bothered me a bit. That is because there is no PTR record for DC01 in a reverse lookup zone, so nslookup cannot resolve the DNS server's own name backwards. It has no effect on forward lookups. The reverse lookup zone configuration is something I should go and look at separately.

```
Test-NetConnection DC01.lumen.local -Port 445
NameResolutionSucceeded : True
TcpTestSucceeded        : True
```

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-35.png?updatedAt=1789013074284)
```
gpupdate /force
Computer Policy update has completed successfully.
User Policy update has completed successfully.
```
![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-36.png?updatedAt=1789013074158)
```
net use
OK    \\DC01\CompanyData    Microsoft Windows Network
```

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-7/LUM7-40.png?updatedAt=1789013073912)
The share opened under the FQDN too, and the internet was still fine. The reason the internet still works when the client only points at internal DNS is that DC01 has forwarders configured. The client asks the internal DNS server, and the DNS server goes and asks on its behalf for anything external.

----------

## What I took away

**Putting a public DNS server on a client is not a shortcut, it is switching off domain functionality.** The internet still works so it looks harmless, but a public resolver knows nothing about the SRV records AD depends on. External names are the DNS server's job, through forwarders.

**If the IP works and the name does not, it is a name resolution problem.** That one line halves the search area.

**Ask the same question to a different server and compare the answers.** `nslookup name serverIP` lets you change who you are asking. If the answers differ, the fault is on the client, not the server.

**Read the nslookup header before the result.** The `Server:` line is "who am I asking right now".

**How something succeeded matters more than the fact that it succeeded.** `ping DC01` worked, but it answered as `DC01.local` from a `fe80::` address. If I had not read the output to the end I would have walked away saying "DNS looks fine".

**Test domain faults with the fully qualified name.** A short name working is not evidence that DNS is healthy. It just means a fallback path is covering for it.

**gpresult is the past, gpupdate is the present.** Do not look at a cached result and call the system healthy.

**On DHCP, restore the method rather than the value.** `-ResetServerAddresses`.

**DHCP enabled plus a manual DNS entry means someone overrode it by hand.** The server is handing out correct values and only this one machine differs, which lines up with a blast radius of one user.

**Read CategoryInfo first on any red error.** PermissionDenied and ObjectNotFound need completely different responses.

**Insufficient privilege does not show up the same way twice.** A GPO gives you a popup, the CLI gives you red text, the GUI just greys things out. The last one is the most confusing.

**On any intermittent "some things work and some things do not" report, start with `ipconfig /all`.** It takes ten seconds and would have caught this one immediately.

----------

## Next

Confirmed there is no PTR record in a reverse lookup zone. That looks like a decent subject for the next networking ticket. WinRM was also switched off, which stopped me running anything remotely, and turning that on through GPO is probably a ticket of its own.

----------