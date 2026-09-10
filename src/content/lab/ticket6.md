---
title: Ticket6 - Account Deactivation and Access Revocation
description: "On Thursday September 3, an offboarding request came in from HR. Olivia Tremblay, the account I had created back in LUM-5, was finishing her contract, and they wanted the account disabled and her access revoked at end of business on Friday."
pubDate: 2026-09-03
heroImage: "https://ik.imagekit.io/stephanie/Ticket-6/LUM6-09.png?updatedAt=1788480383335"
badge: "IT Support"
tags:

  - Service Request
  - mployee Offboarding
---


## Account Deactivation and Access Revocation

**Ticket:** LUM-6 </br>
**Date:** 2026-09-04 (request received 2026-09-03) </br>
**Category:** Service Request / Employee Offboarding </br>
**Environment:** Windows Server 2022 (DC01), Windows 11 (CLIENT01), LUMEN.LOCAL </br>

## The request

On Thursday September 3, an offboarding request came in from HR. Olivia Tremblay, the account I had created back in LUM-5, was finishing her contract, and they wanted the account disabled and her access revoked at end of business on Friday.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-6/LUM6-02.png?updatedAt=1788480383121)

There was a condition attached. Do not do it early, do it at the end of her last working day, because she was working the full day. There was also a note that if she left any files in the Sales shared folder, Michael Lee needed to be able to reach them.

Since this is a lab, I compressed the gap between onboarding and offboarding down to a few days. In reality it would be separated by the length of the contract, but I wanted to see the whole life of a single account, from creation to closure, as one continuous thread.

I set this one as P2. LUM-5 had been P3, so I thought about what actually made the difference. If onboarding runs late, someone is mildly inconvenienced. If offboarding runs late, a departed employee still has live access. And doing it too early means someone who is still working cannot sign in. The window for this work was narrow and specific, and that is what raised the priority.

## Doing the work

Before starting I created a Disabled Users OU, placed directly under the domain root, with no GPO linked to it. There is no reason for policy to apply to a dormant account, and the absence of any link is itself a statement that the account is isolated.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-6/LUM6-03.png?updatedAt=1788480383276)

The next step turned out to be the most important one in this ticket, and I nearly skipped straight past it.

```powershell
Get-ADUser otremblay -Properties MemberOf, Description |
  Select-Object SamAccountName, DistinguishedName, Enabled, Description

Get-ADPrincipalGroupMembership otremblay | Select-Object Name
```

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-6/LUM6-04.png?updatedAt=1788480383264)

Record the current state before touching anything. Once you pull someone out of a group, there is no way left to find out which groups they were in. If somebody asks later what access this person actually had, this record is the only answer that exists. I pasted the output straight into the ticket as an internal note.

With that captured, I worked through the changes in order.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-6/LUM6-05.png?updatedAt=1788480383245)

PowerShell is not the only way. In Active Directory Users and Computers you can right click the user account and disable it in a couple of clicks.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-6/LUM6-07.png?updatedAt=1788480383291)

```powershell
Disable-ADAccount -Identity otremblay

Set-ADAccountPassword -Identity otremblay -Reset `
  -NewPassword (Read-Host "New random password" -AsSecureString)

Remove-ADGroupMember -Identity "Sales-Users" -Members otremblay -Confirm:$false

Move-ADObject -Identity (Get-ADUser otremblay).DistinguishedName `
  -TargetPath "OU=Disabled Users,DC=lumen,DC=local"

Set-ADUser otremblay -Description "Disabled 2026-09-04 per LUM-6. Contract end. Retain until 2026-12-03."
```

Two choices matter here. The account was disabled rather than deleted, and the password was reset to a random value. The reset is there to invalidate any credential that might be saved somewhere I cannot see.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-6/LUM6-08.png?updatedAt=1788480383258)

Moving the account to another OU had a side effect worth noting. It also fell out of scope for Sales-Desktop-Standards, the GPO linked to the Sales OU. That was the intended outcome, but it is the kind of thing that belongs in the work log rather than being left implicit.

## Verifying the revocation

This is where I hit a wall I had not seen coming.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-6/LUM6-09.png?updatedAt=1788480383335)

I went to confirm that access was blocked, and realized I could not sign in as the account to open the folder, because the account was disabled. Authentication fails first, so you never get far enough to test authorization at all.

Thinking it through, there were two separate layers to verify.

<table><thead><tr><th>Layer</th><th>What blocks it</th><th>How to verify</th></tr></thead><tbody><tr><td>Authentication</td><td>Account is disabled</td><td>runas logon attempt</td></tr><tr><td>Authorization</td><td>Removed from Sales-Users</td><td>Effective Access</td></tr></tbody></table>

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-6/LUM6-12.png?updatedAt=1788480383310)

For authentication I used runas, which lets you attempt a logon as another account while keeping your own administrator session open.

```
runas /user:LUMEN\otremblay cmd
```

The result was not what I expected. I was waiting for error 1331, the one that says the account is disabled. Instead I got this.

```
RUNAS ERROR: Unable to run - cmd
1327: Account restrictions are preventing this user from signing in.
```

It took me a while to work out why. It was because I had typed a random password. runas checks the credentials first and looks at the account state second. The password was wrong, so it failed at step one, and instead of a specific reason it returned the broader 1327.

What I found interesting is that this is deliberate. Telling someone who does not even know the password that the account exists but is currently disabled would hand information to an attacker. Windows answers vaguely on purpose.

<table><thead><tr><th>Code</th><th>Meaning</th><th>When it appears</th></tr></thead><tbody><tr><td>1326</td><td>Bad username or password</td><td>Password wrong only</td></tr><tr><td>1327</td><td>Account restrictions prevent sign-in</td><td>Password wrong and account also restricted</td></tr><tr><td>1331</td><td>Account is currently disabled</td><td>Password correct</td></tr></tbody></table>

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-6/LUM6-11.png?updatedAt=1788480383561)

To be sure of what I was looking at, I set up a control. Running the same command against mlee, an active account, gave a different result. That confirmed the block was specific to otremblay and not something wrong with CLIENT01 itself.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-6/LUM6-13.png?updatedAt=1788480383280)

For authorization I used Effective Access. On the Sales folder, otremblay now had nothing.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-6/LUM5-14.png)

That was the direct opposite of the Modify result from LUM-5.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-6/LUM6-19.png)

```powershell
icacls C:\CompanyData\Sales
```

At the same time I confirmed that LUMEN\Sales-Users was still sitting in the folder ACL, untouched. Access was removed by taking the person out of the group, not by editing the folder permissions. That is the correct way round. Editing a folder ACL because one person left is how you end up with a pile of exception rules that nobody can explain a year later.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-6/LUM6-17.png?updatedAt=1788480383140)

The last thing was to check for data left behind. In File Explorer I searched the Sales folder to list everything recursively, right clicked the column header, added the Owner column from More, and sorted by owner. There were no files owned by otremblay.

I did consider skipping this step since there was nothing there. I did it anyway. "No orphaned data" has to be something you confirmed, not something you assumed. An empty result is still evidence.

## Closing out

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-6/LUM6-10.png?updatedAt=1788480383217)

With the work done I checked the final state. Enabled was False, the DN pointed at the Disabled Users OU, and Domain Users was the only group left. The Description field carried the date, the ticket number, and the retention deadline.

In the reply to HR I explained why the account was not deleted. Deleting it would sever the ownership records and audit history for anything she created. I also included the note that it is scheduled for review in 90 days.

Michael Lee already had Modify on the Sales folder, so nothing was needed on his side.

## What I took away

Why disable instead of delete? The answer is the SID. File ownership, mailbox delegation and audit logs are all tied to it. Delete the account and those links break, leaving unresolvable entries like Account Unknown behind. Recreating the account with the same name does not fix it, because the new account gets a different SID.

Record what you are about to revoke before you revoke it. Do it the other way round and you can neither reverse the change nor answer an audit question about it. The single most practical thing in this ticket was that one step.

Disabling is not the same as blocking immediately. A Kerberos ticket that has already been issued stays valid until it expires, and the default TGT lifetime is 10 hours, so an existing session can survive for a while after the account is switched off. If there is an active session, it has to be logged off, not assumed dead.

The moment I learned the most from was the error code that did not match what I expected. When 1327 came back my first thought was that I had done something wrong. Digging into it, I found a deliberate design decision to avoid leaking information. Changing the command as soon as the screen looks unfamiliar teaches you far less than working out why it looks that way.

Running LUM-5 and LUM-6 back to back gave me the full arc of a single account, from creation to closure. Onboarding turned on confirming that the things which should be blocked really were blocked. Offboarding turned on recording the state before revoking it. Two halves of the same story.