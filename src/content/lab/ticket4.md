---
title: Ticket4 - The GPO Applied, But the Drive Never Mapped
description: "An HR user (`skim`) signed in to her workstation, CLIENT01, and the `H:` drive holding the candidate application files was nowhere to be found. It had worked the day before, and a coworker on the IT team said the same drive showed up fine at their desk. She had already restarted twice before opening the ticket."
pubDate: 2026-08-26
heroImage: "https://ik.imagekit.io/stephanie/Ticket-4/LUM4-22.png?updatedAt=1788029366442"
badge: "IT Support"
tags:

  - Active Directory
  - Group Policy Preferences
---


## The GPO Applied, But the Drive Never Mapped

**Ticket:** LUM-4 · Lumen Systems IT Service Desk </br>
**Date:** 2026-08-26</br>
 **Category:** Active Directory / Group Policy Preferences </br>
 **Environment:** Windows Server 2022 (DC01), Windows 11 (CLIENT01), domain `LUMEN.LOCAL`</br>

----------

## Situation

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-4/LUM4-01.png?updatedAt=1788029365717)
An HR user (`skim`) signed in to her workstation, CLIENT01, and the `H:` drive holding the candidate application files was nowhere to be found. It had worked the day before, and a coworker on the IT team said the same drive showed up fine at their desk. She had already restarted twice before opening the ticket.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-4/LUM4-11.png?updatedAt=1788029366276)
The single line she wrote into the ticket herself turned out to be the most valuable piece of information in it: typing the UNC path directly into the address bar opened the folder without any trouble.

----------

## Symptoms

No error message was ever shown to the user. The drive simply was not there. That silence is what defines this class of failure.

Checking the mapping status in the user's session returns nothing at all.
![enter image description here](https://ik.imagekit.io/stephanie/Ticket-4/LUM4-09.png?updatedAt=1788029366338)

```powershell
C:\Users\skim>net use
New connections will be remembered.

There are no entries in the list.
```
No error, no output. That means zero active mappings.
![enter image description here](https://ik.imagekit.io/stephanie/Ticket-4/LUM4-22.png?updatedAt=1788029366442)

Get-SmbMapping returns an empty result as well.
Same meaning: the number of mappings is zero. There is not a single one.

Direct access to the share, on the other hand, was fine.
![enter image description here](https://ik.imagekit.io/stephanie/Ticket-4/LUM4-10.png?updatedAt=1788029366199)

```powershell
PS C:\Users\skim> Test-Path "\\DC01\CompanyData\HR"
True
```

The folder exists, the server is reachable, and the user can read it. The only thing missing is the drive letter.

----------

## Investigation

### Ruling out the entire permissions layer first

The reporter's own observation did half the diagnostic work. If `\\DC01\CompanyData\HR` opens, then everything below is already proven healthy.

-   Network connectivity between CLIENT01 and DC01
-   DNS resolution of the `DC01` hostname
-   SMB share-level permissions
-   NTFS ACLs on the target folder

One test eliminates four candidate causes. What remains is narrow: the mechanism that is supposed to attach the letter `H:` to that path, and nothing else. In this environment that mechanism is Group Policy Preferences (GPP) Drive Maps.


### Trying to save the report to a path that did not exist

The next attempt failed too, this time for a completely unrelated reason.

```
PS C:\WINDOWS\system32> gpresult /h C:\Temp\LUM-4-gpresult.html
ERROR: The system cannot find the path specified.
```

`gpresult` will create the file but it will not create the directory. `C:\Temp` did not exist on this machine. The word "path" in the error message means exactly that, the folder, not a missing drive.

Saving to a location that always exists avoids the problem entirely.

```powershell
gpresult /h "$env:USERPROFILE\Desktop\LUM-4-gpresult.html"
```

There is one more thing here that matters more than it looks. `gpresult` reports the policy result for **whoever ran the command**. If you launch an elevated prompt and enter domain admin credentials, you get that admin's RSoP, not the user's. For this ticket it had to be run inside skim's own session at standard privilege. The two seconds it takes to type `whoami` before running it are well spent.

### The report said the GPO had applied

This is where the heart of the ticket is. The RSoP report came back like this.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-4/LUM4-12.png?updatedAt=1788029365979)

<table> <thead> <tr> <th>Check</th> <th>Result</th> </tr> </thead> <tbody> <tr> <td>Applied GPOs (User Configuration)</td> <td><code>HR-Drive-Mapping</code> <strong>present</strong></td> </tr> <tr> <td>Denied GPOs</td> <td>not listed</td> </tr> <tr> <td>User OU location</td> <td><code>OU=HR,DC=lumen,DC=local</code> (correct)</td> </tr> </tbody> </table>

The GPO was applying. It was not filtered out, not blocked, and the user object was in the right OU. Every check that would normally explain a Group Policy failure came back clean.

This is the point where the diagnostic layer has to change. "Did the GPO reach the user?" has been answered. Yes. The remaining question is a different one: "did the individual Preference item inside that GPO actually execute?"

### Checking whether the GPO item itself ran

The place to ask that question is the Group Policy operational log.
![enter image description here](https://ik.imagekit.io/stephanie/Ticket-4/LUM4-15.png?updatedAt=1788029366183)

```powershell
Get-WinEvent -LogName "Microsoft-Windows-GroupPolicy/Operational" -MaxEvents 40 |
    Select-Object TimeCreated, Id, Message | Format-List
```

The policy processing cycle was recorded as successful, but there was no record of the Drive Maps item being applied. A GPO arriving and a Preference item running are two separate events, and only the first one had happened here.

That pattern points to item-level targeting. A GPP item can carry its own condition, and that condition is evaluated on the client **after** the GPO has been delivered. If the condition is false, the item is skipped silently. No error, no log entry, and no user-visible symptom beyond the missing drive.

### Verifying the targeting condition

The Drive Maps item had a Security Group condition on `LUMEN\HR-Drive-Users`. 
![enter image description here](https://ik.imagekit.io/stephanie/Ticket-4/LUM4-08.png?updatedAt=1788029365991)

A Security Group condition is evaluated against the user's access token, so the token is what needs to be inspected.
![enter image description here](https://ik.imagekit.io/stephanie/Ticket-4/LUM4-17.png?updatedAt=1788029366220)
```
PS C:\Users\skim> whoami /groups

GROUP INFORMATION
-----------------

Group Name                              Type              SID
======================================= ================= ==============================
Everyone                                Well-known group  S-1-1-0
BUILTIN\Users                           Alias             S-1-5-32-545
NT AUTHORITY\INTERACTIVE                Well-known group  S-1-5-4
CONSOLE LOGON                           Well-known group  S-1-2-1
NT AUTHORITY\Authenticated Users        Well-known group  S-1-5-11
NT AUTHORITY\This Organization          Well-known group  S-1-5-15
LOCAL                                   Well-known group  S-1-2-0

```

`LUMEN\HR-Drive-Users` is not there.
`LUMEN\HR-Users` is, and mistaking that one for a match is exactly the trap to avoid.

Cross-checked on the domain controller.
The next thing to confirm is who is actually in the HR drive group, and whether it has any members at all.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-4/LUM4-18.png?updatedAt=1788029366240)

```powershell
PS C:\> Get-ADGroupMember -Identity "HR-Drive-Users"
PS C:\>
```

Empty. The group existed, but it had no members.
That pins down the cause.

**Root cause:** the user was not a member of the security group used by the Drive Maps item's item-level targeting. The condition evaluated to false and the item was skipped. The GPO container itself applied normally, and that is precisely why the Applied GPOs list could not surface this fault.

----------

## Resolution

### Adding the user to the HR drive group
![enter image description here](https://ik.imagekit.io/stephanie/Ticket-4/LUM4-19.png?updatedAt=1788029365951)
GUI path (DC01): `dsa.msc` → HR OU → double-click `HR-Drive-Users` → Members tab → Add → `skim` → Check Names → OK

PowerShell:

```powershell
Add-ADGroupMember -Identity "HR-Drive-Users" -Members skim
Get-ADGroupMember -Identity "HR-Drive-Users" | Select-Object Name, SamAccountName
```
![enter image description here](https://ik.imagekit.io/stephanie/Ticket-4/LUM4-20.png?updatedAt=1788029366423)

This confirms the member was added correctly.

### Dead end 3: gpupdate was not enough

The obvious next step did not work.

```cmd
gpupdate /force
```

Policy refreshed successfully. The drive still did not appear.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-4/LUM4-21.png?updatedAt=1788029366229)
Here is why, and it is worth knowing, because this is the number one reason users come back saying a fix did not work.

When you sign in, Windows builds a list of every group you belong to and attaches it to your session. That list is your access token. It is built **once, at sign-in**, and Windows never updates it while you stay signed in.

So adding skim to `HR-Drive-Users` changed Active Directory, but it did not change the token already sitting in her open session. Her token still had no `HR-Drive-Users` in it.

Item-level targeting checks the token, not Active Directory. The token was out of date, so the condition was still false, and the Drive Maps item was still skipped. `gpupdate` re-downloads policy, but it does not rebuild the token, so there was nothing it could fix.

### Sign out, then sign back in

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-4/LUM4-26.png)
This rebuilds the token. After signing back in:

```
PS C:\Users\skim> whoami /groups

Group Name                              Type    SID
======================================= ======= ====================================================
LUMEN\HR-Drive-Users                    Group   S-1-5-21-409842420-2678424927-1604956302-1118

```

The group is now in the token. The drive checked out too.
![enter image description here](https://ik.imagekit.io/stephanie/Ticket-4/LUM4-24.png?updatedAt=1788029365988)
```
PS C:\> Get-SmbMapping

Status Local Path Remote Path
------ ---------- -----------
OK     H:         \\DC01\CompanyData\HR

```
![enter image description here](https://ik.imagekit.io/stephanie/Ticket-4/LUM4-25.png?updatedAt=1788029365967)
Verified in File Explorer as well. `H:` appears under This PC, and reading from and writing to the folder both work. The reporter confirmed it as well.

----------

## Takeaways

**A GPO applying and a setting inside that GPO executing are two different claims.** The Applied GPOs list in `gpresult` only confirms that the policy container was delivered to the user. It says nothing about whether a Group Policy Preferences item inside it actually ran, because item-level targeting is evaluated on the client **after** delivery. If a drive, printer, or shortcut is missing while `gpresult` looks perfectly healthy, the next place to look is the Preferences layer, not the GPO layer.

**Group membership changes require a new logon.** `gpupdate /force` refreshes policy but does not rebuild the access token. Any condition that depends on group membership will keep evaluating against the old token until the user signs out and back in. On a real service desk it is worth telling the user this up front. Otherwise you get a follow-up saying the fix did not work.

**The reporter's own observation cuts down the search space.** The single sentence "it opens if I type the address directly" removed network, DNS, share permissions, and NTFS ACLs from the candidate list. Reading the ticket carefully before touching anything is faster than working up from the bottom of the stack.

### First thing to check next time

If a mapped drive is missing but the UNC path opens fine, skip the permissions layer entirely and go straight down this list.

1.  Run `gpresult /h` at standard privilege inside the user's own session, and check the Applied and Denied GPO lists
2.  If the GPO is applied, open it in `gpmc.msc` and check the **Common** tab of the Preference item for item-level targeting
3.  Run `whoami /groups` in the user's session to verify whether the targeting condition can even be true
4.  If you change group membership, sign the user out and back in, and state that in the resolution comment

Each step cuts the remaining candidates roughly in half, and the whole sequence takes under ten minutes.