---
title: Ticket3 - Linked correctly, filtered correctly, and still never applied
description: " a new hire in Sales, reported that the standard desktop settings rolled out to her department hadn't reached her PC. Michael Lee (`mlee`), on the same team, said his had updated the previous afternoon without any issue."
pubDate: 2026-08-25
heroImage: "https://ik.imagekit.io/stephanie/Ticket-3/LUM3-11.png?updatedAt=1788029248862"
badge: "IT Support"
tags:

  - Active Directory
  - Group Policy
  - GPO Scope 
---


## Linked correctly, filtered correctly, and still never applied

**Ticket:** LUM-3 · Lumen Systems IT Service Desk </br>
**Date:** 2026-08-25 </br>
**Category:** Group Policy / GPO Scope </br>
**Environment:** Windows Server 2022 (DC01), Windows 11 (CLIENT01), domain `LUMEN.LOCAL` </br>
**Root cause:** Account provisioning error. The user object sat outside the OU the GPO was linked to. </br>

---

## The situation

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-3/LUM3-24.png?updatedAt=1788029248460)
Jessica Park (`jpark`), a new hire in Sales, reported that the standard desktop settings rolled out to her department hadn't reached her PC. Michael Lee (`mlee`), on the same team, said his had updated the previous afternoon without any issue.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-3/LUM3-01.png?updatedAt=1788029248483)
The policy in question was `Sales-Desktop-Standards`, a User Configuration GPO linked to the Sales OU. Its only setting was **Remove Recycle Bin icon from desktop**, which made verification simple. Either the icon is there or it isn't.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-3/LUM3-02.png?updatedAt=1788029248986)
This is what an account with the Sales policy correctly applied should look like. No Recycle Bin.

The fact that only one person was affected stood out to me from the start. It turned out to be the single most useful piece of information in this ticket.

----------

## The symptoms

I signed in to CLIENT01 as `jpark` and the Recycle Bin icon was still sitting there. So I checked the policy result.
![enter image description here](https://ik.imagekit.io/stephanie/Ticket-3/LUM3-11.png?updatedAt=1788029248862)

Output like this tends to look the same no matter what you're looking at, but put it side by side with another account in the same OU and you can feel that something about Jessica's account is off.

```
C:\Users\jpark>gpresult /r /scope:user

USER SETTINGS
--------------
    CN=Jessica Park,CN=Users,DC=lumen,DC=local
    Last time Group Policy was applied: 2026-08-25 at 12:54:20 PM
    Group Policy was applied from:      DC01.lumen.local
    Domain Name:                        LUMEN

    Applied Group Policy Objects
    -----------------------------
        N/A

    The following GPOs were not applied because they were filtered out
    -------------------------------------------------------------------
        Local Group Policy
            Filtering:  Not Applied (Empty)

```

Running the same command as `mlee` gave a completely different result.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-3/LUM3-16.png?updatedAt=1788029249037)

```
C:\Users\mlee>gpresult /r /scope:user

USER SETTINGS
--------------
    CN=Michael Lee,OU=Sales,DC=lumen,DC=local

    Applied Group Policy Objects
    -----------------------------
        Sales-Desktop-Standards

```

There it is. Look closely at the two accounts and Jessica's is missing `OU=Sales` entirely.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-3/LUM3-12.png?updatedAt=1788029249048)
Two things jumped out. `Sales-Desktop-Standards` wasn't in Jessica's applied list, and it **wasn't in the denied list either.** The name didn't appear anywhere in the output.

Here's the problem section:

```
Applied Group Policy Objects 
----------------------------- 
N/A 
The following GPOs were not applied because they were filtered out 
------------------------------------------------------------------- 
Local Group Policy Filtering: Not Applied (Empty)
```

So the next job was figuring out where this went wrong.

----------

## Narrowing down the cause

There are three reasons a GPO fails to reach a user, and they need to be checked in order.

1.  The GPO isn't linked to the OU, or the link is disabled
2.  The GPO is linked, but Security Filtering blocks the object from reading it
3.  The object isn't inside the OU the GPO is linked to

### Ruling out a link problem

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-3/LUM3-17.png?updatedAt=1788029248649)
`mlee` is in the same OU and received the policy without any trouble. If the link had been removed, neither of them would have gotten it. **Cause 1 eliminated.**

Testing another account in the same scope takes about two minutes, and doing it split the problem into "the policy is broken" versus "this account is broken." Those two lead to completely different investigations.

Looking at mlee's account, there's nothing wrong with the policy.

### Ruling out Security Filtering

I ended up checking this with the HTML report, and hit two dead ends along the way.

Nothing ever goes smoothly on the first try, but honestly that might be for the best.

**Dead end 1: Access is denied**

```
C:\Users\jpark>gpresult /h C:\gpreport.html
ERROR: Access is denied.
```

The command wasn't the problem, the save location was. Standard users can't write to the root of `C:\`.

My first thought was "it's a permissions problem, so let me rerun it as administrator," and that was the wrong call. `gpresult` reports on **the account that runs it.** Run it from an elevated prompt and you get the Administrator's policy result. The target of the investigation quietly changes on you. What needed fixing was the output path, not the privilege level.

**Dead end 2: The system cannot find the path specified**

Short version, this failed too. `%USERPROFILE%` style syntax is Command Prompt only, so of course it errors out in PowerShell.

```
PS C:\WINDOWS\system32> gpresult /h %USERPROFILE%\gpreport.html
ERROR: The system cannot find the path specified.
```

I changed the path and it kept failing. Tried it a few different ways with the same result. The answer showed up in the next command.

```
PS C:\WINDOWS\system32> dir %USERPROFILE%
dir : Cannot find path 'C:\WINDOWS\system32\%USERPROFILE%' because it does not exist.
```

Well, no wonder it doesn't exist.

Here's what worked:

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-3/LUM3-13.png?updatedAt=1788029248735)
Written this way, it runs fine.

`%USERPROFILE%` was sitting in the path as literal text. I'd been using the wrong shell. `%VARIABLE%` is Command Prompt syntax, and PowerShell passes it through as a plain string instead of expanding it.

In PowerShell you use `$env:USERPROFILE`.

The way to sidestep the syntax difference entirely is to change directory first and just use a bare filename. That works identically in both shells.

**What the report showed**

I opened Jessica's report.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-3/LUM3-14.png?updatedAt=1788029248633)

```
Group Policy Objects
  Applied GPOs
      (none)
  Denied GPOs
      Local Group Policy [LocalGPO]
          Link Location    Local
          Security Filters
          Reason Denied    Empty
```

At first glance I thought the report was just empty. `Sales-Desktop-Standards` wasn't even in the denied list, so I assumed there was no information there. That turned out to be the key observation.

Normally a GPO blocked by Security Filtering **does show up under Denied GPOs**, with `Access Denied (Security Filtering)` as the reason. It got evaluated and rejected.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-3/LUM3-15.png?updatedAt=1788029248625)
But Jessica's report has nothing at all under Applied GPOs. In mlee's report above, you can see the applied GPO listed underneath.

If the user object isn't inside the linked OU, the GPO **is never evaluated in the first place**, so there's no rejection reason to record.

"Blocked" and "out of scope" look nothing alike once you know where to look. **Cause 2 eliminated.**

### Confirming the actual cause

Back to the first line of the `gpresult` output.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-3/LUM3-12.png?updatedAt=1788029249048)

```
CN=Jessica Park,CN=Users,DC=lumen,DC=local
```

That suspicious line I'd noticed earlier.

Checking in `dsa.msc`, the account was in the default **Users** container, not the Sales OU.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-3/LUM3-10.png?updatedAt=1788029248616)

`Users` is a container, not an OU. Group Policy can't be linked to a container. Right-click Users in `dsa.msc` and the "Create a GPO in this domain, and Link it here" option that every OU has simply isn't there. I clicked through to confirm it myself. Which means no department policy could ever have reached this account, no matter how the GPO was configured.

Checking Group Policy Management on DC01, the GPO itself had been fine the whole time. The link to the Sales OU was enabled, and Authenticated Users had Read and Apply Group Policy under Security Filtering. There was nothing wrong with the policy.

----------

## The fix

**GUI**

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-3/LUM3-18.png?updatedAt=1788029248786)

1.  Run `dsa.msc` on DC01
2.  Expand `lumen.local` and select the **Users** container
3.  Right-click **Jessica Park** and choose **Move...**
4.  Select the **Sales** OU and confirm

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-3/LUM3-19.png?updatedAt=1788029248781)
Jessica, moved successfully.

**PowerShell**

```powershell
Move-ADObject -Identity "CN=Jessica Park,CN=Users,DC=lumen,DC=local" `
              -TargetPath "OU=Sales,DC=lumen,DC=local"
```

**Verifying on CLIENT01**

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-3/LUM3-20.png?updatedAt=1788029248840)

```
gpupdate /force
```

This step is essential. Skip it and no matter how much you change the policy on the server, the client isn't going to update itself out of nowhere.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-3/LUM3-21.png?updatedAt=1788029248968)
Then I signed out and back in. `gpupdate` alone wasn't enough. The user object had moved to a different OU, so the client needed a fresh logon to rebuild the user's policy scope.

```
C:\Users\jpark>gpresult /r /scope:user

USER SETTINGS
--------------
    CN=Jessica Park,OU=Sales,DC=lumen,DC=local

    Applied Group Policy Objects
    -----------------------------
        Sales-Desktop-Standards

```

The last check was on the desktop, not in the report. The Recycle Bin icon was gone.

I'd assumed that step was a formality, but it wasn't. During the earlier comparison test, `mlee` had `Sales-Desktop-Standards` showing under Applied GPOs while the Recycle Bin was still sitting on his desktop. The shell had already drawn the desktop before the policy took effect. `gpresult` reporting success and the setting actually showing up on screen are two different claims, and the second one is what the user is asking about.

----------

## What I took away

**Scope of impact narrows the cause before you open a single tool.** One person affected means you look at that person's object. OU placement, group membership, profile. A whole department affected means you look at the policy. Link, filtering, settings. Just reading the "how many people are affected" field properly saves a lot of time.

**Check the three causes in a fixed order.** Link status, Security Filtering, then the object's OU placement. All three produce the identical symptom of "nothing changed on my PC," so guessing is actually slower than working down the list.

**Absence from the denied list is evidence too.** Blocked by filtering means the name shows up with a reason. Out of scope means it doesn't show up at all. Once I understood that difference, a report that had looked empty turned into a result that eliminated a cause.

**`gpresult` reports on the account that runs it.** Run it in the affected user's own session at standard privilege. Elevating to get around a file permission error changes what you're measuring.

**Verify the setting, not the report.** A GPO appearing under Applied GPOs means Group Policy processed it. It doesn't mean the user is looking at a changed desktop.

**Check which shell you're in.** `%VAR%` is Command Prompt, `$env:VAR` is PowerShell. When a path fails for no apparent reason, look at the prompt before you look at the path.

----------

## Preventing a repeat

It looks obvious in hindsight, but this is one of those mistakes that actually happens a lot.

This account seemed completely normal from the outside. Login worked, mail worked, file shares worked. It only surfaced weeks later when a department policy went out and one person got left behind. That delay is the real danger with this class of error.

I added an OU placement check to the new hire account creation checklist. Creating an account with `net user`, or joining a domain without specifying a target, drops the object into the default `Users` or `Computers` container, where no Group Policy can ever reach it. For computer objects, `redircmp` should be used to redirect the default location to a linked OU.