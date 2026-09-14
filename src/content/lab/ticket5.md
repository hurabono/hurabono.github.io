---
title: Ticket5 - New Account Provisioning and Access Verification
description: "An onboarding request came in from Sarah Kim in HR. Olivia Tremblay was joining the Sales team, starting Monday August 31. Contract position, reporting to Michael Lee. She needed the same access the rest of the Sales team has, including the Sales shared folder."
pubDate: 2026-08-31
heroImage: "https://ik.imagekit.io/stephanie/Ticket-5/LUM5-03.png?updatedAt=1788480123928"
badge: "IT Support"
tags:

  - Service Request
  - Employee Onboarding
---


## New Account Provisioning and Access Verification

**Ticket:** LUM-5 </br>
**Date:** 2026-08-30 </br>
**Category:** Service Request / Employee Onboarding </br>
**Environment:** Windows Server 2022 (DC01), Windows 11 (CLIENT01), LUMEN.LOCAL </br>

## The request

An onboarding request came in from Sarah Kim in HR. Olivia Tremblay was joining the Sales team, starting Monday August 31. Contract position, reporting to Michael Lee. She needed the same access the rest of the Sales team has, including the Sales shared folder.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-5/LUM5-01.png)

One line in the request stood out. The temporary password should not be written in the ticket, and should go to Michael directly instead. At first I wondered why anyone would bother specifying that. Then it clicked: a ticket comment is a permanent record that plenty of people can open later. It was a reasonable thing to ask for.

Everything I had worked on so far, LUM-1 through LUM-4, had been an incident. Something was broken and I had to find it and fix it. This one was the opposite. Nothing was broken. I was building something from scratch, and that turned out to be a different kind of work.

## Building the account

I started by confirming values instead of assuming them. Guessing an OU path or a group name and typing it straight into a command is a habit worth avoiding.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-5/LUM5-02.png?updatedAt=1788480123946)

```powershell
Get-ADOrganizationalUnit -Filter 'Name -eq "Sales"' |
  Select-Object DistinguishedName

Get-ADGroup -Filter 'Name -like "*Sales*"' |
  Select-Object Name, DistinguishedName
```

That confirmed `OU=Sales,DC=lumen,DC=local` and `Sales-Users`, so I created the account.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-5/LUM5-03.png?updatedAt=1788480123928)

I followed the same naming standard as the existing accounts, first initial plus last name, so `otremblay`, with the UPN `otremblay@lumen.local`. I filled in Display name, Job title, Department and Manager, and checked "User must change password at next logon."

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-5/LUM5-06.png?updatedAt=1788480123915)

This is where I got stuck for the first time. I wanted a short, easy temporary password, and the server kept rejecting it as not meeting policy requirements. It had uppercase, lowercase and numbers in it, so I could not work out what was missing.

```powershell
Get-ADDefaultDomainPasswordPolicy
```

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-5/LUM5-16.png?updatedAt=1788562467606)

The answer was in the output. `MinPasswordLength` in this domain is 12, not the Windows default of 7. Every value I had tried was shorter than that. The character types were never the problem. It was purely length, and I had been troubleshooting the wrong thing because I never checked the policy before guessing.

I did think about lowering the minimum length to make the lab easier. I decided against it. Turning down a domain security policy for my own convenience leaves a permanent trace in the environment, and it would be a strange thing to explain later. Writing a password that meets the rule was the correct answer.

## Verifying access

Creating the account took about five minutes. Verifying it was the actual job.

My first instinct was to sign in on CLIENT01 as Olivia and click around to see what worked. Then I found out that this is something you do not do in a real environment, for three reasons.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-5/LUM5-07.png?updatedAt=1788480124148)

First, the security log records a sign-in by otremblay when it was really the administrator. That corrupts the audit trail. Second, an administrator who signs in consumes the "change at next logon" prompt, which means the administrator ends up knowing the user's real password. That defeats the whole point of handing the temporary credentials to the manager only. Third, in a lot of organizations, using someone else's account is a policy violation on its own.

So I changed approach and verified from the server side, without signing in as anyone.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-5/LUM5-14.png?updatedAt=1788562467680)

On the Sales folder I went to Properties > Security > Advanced > Effective Access and selected otremblay. Modify came back with a green check. Without a single logon, Windows had calculated exactly what this account would be able to do.

The next part mattered more. I ran the same check against the Finance, HR and IT folders. Every permission came back as a grey X. No access.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-5/LUM5-15.png?updatedAt=1788562467644)
This is the Effective Access tab under Advanced Security Settings for the IT folder. Running the same Effective Access check on the other folders is how I confirmed whether Olivia could reach the IT folder.
<table><thead><tr><th>Folder</th><th>Effective Access for otremblay</th><th>Expected</th></tr></thead><tbody><tr><td>Sales</td><td>Modify</td><td>Allowed</td></tr><tr><td>Finance</td><td>None</td><td>Blocked</td></tr><tr><td>HR</td><td>None</td><td>Blocked</td></tr><tr><td>IT</td><td>None</td><td>Blocked</td></tr></tbody></table>

From an administrator session the Finance folder opens without any trouble, because Administrators sits in the folder ACL with Full Control. Which means clicking into a folder as an administrator tells you nothing at all about what a user can see. It took me a while to really absorb that difference.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-5/LUM5-12.png?updatedAt=1788480124124)

There was still the GPO to confirm. I needed to know whether Sales-Desktop-Standards, the policy from LUM-3, would reach this account. But `gpresult` reports what has already happened, and this account had never signed in, so it had nothing to report.

I found out that Group Policy Modeling in the Group Policy Management console can simulate this before a first logon. I ended up confirming it at the real first sign-in this time, but that tool is what I plan to use on the next onboarding.

## Handover and closure

With the account ready, the temporary credentials went to Michael Lee directly. Nothing was written into the ticket.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-5/LUM5-08.png?updatedAt=1788480124134)

On her first morning I connected remotely while Olivia signed in herself. The password change prompt appeared as expected and she set her own password. In the Sales folder she created a test file and deleted it, since read access and write access are two different questions and both needed checking.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-5/LUM5-11.png?updatedAt=1788480124248)

On the same screen, opening the Finance folder returned Access Denied. What the server had calculated in advance matched what the user actually saw.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-5/LUM5-13.png?updatedAt=1788480124154)

`gpresult /r` ran in her own session too. Sales-Desktop-Standards was listed under applied policies. I ran it from a standard prompt rather than an elevated one, since an elevated prompt can report results for a different account.

## What I took away

Creating the account was five minutes. Everything after that was verification, and the verification was the real work.

The step that mattered most was checking that the folders she should not reach were actually blocked. At first I did not see the point. She was only added to the Sales group, so of course the rest would be closed. But unintended permissions do happen, through nested group membership, an inherited ACE from a parent folder, or a policy scoped more broadly than someone intended. If you never check, you simply never find out.

The other thing was that the administrator's view and the user's view are not the same view. No amount of clicking around as an administrator tells you what a user experiences. That is exactly why a tool like Effective Access exists.

I also stopped assuming when troubleshooting the password policy. I spent longer than I should have guessing at character requirements when one command would have shown me the real rule. Reading the configuration is faster than reasoning about it.

PowerShell looks different to me now as well. I had assumed I would need to memorize a long list of commands, until I noticed they all follow the same Verb-AD-Noun shape. Get, Set, New, Remove, Add, Enable, Disable, Move. That is close to the whole vocabulary. When I do not know something, `Get-Command` and `Get-Help -Examples` will tell me. It is less about memorizing and more about assembling.