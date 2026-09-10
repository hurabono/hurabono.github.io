---
title: Ticket2 - when company Account Locked Out
description: "This was a full solo run at an account lockout ticket, start to finish. My original plan was to just spin up a test account (tuser) and lock that instead."
pubDate: 2026-08-24
heroImage: "https://ik.imagekit.io/stephanie/Ticket-2/LUM2-14.png?updatedAt=1788029214929"
badge: "IT Support"
tags:

  - Active Directory
  - Account Lockout
  - EventLog 
---


## Locked Out Right Before a 10 AM Meeting: The Full LUM-2 Story

**Ticket:** LUM-2 · Lumen Systems IT Service Desk  </br>
**Date:** 2026-08-24  </br>
**Category:** Active Directory / Account Lockout  </br>
**Environment:** Windows Server 2022 (DC01), Windows 11 (CLIENT01), domain `LUMEN.LOCAL`</br>

---

## Original Ticket (Portal Form)

**Portal form:** Report a login problem **Type:** Incident · **Priority:** P2 · **Component:** Active Directory

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-38.png?updatedAt=1788029214673)

<table> <thead> <tr><th>Field</th><th>Value</th></tr> </thead> <tbody> <tr><td>Summary</td><td>Locked out of my account and I can't get back in</td></tr> <tr><td>Affected services</td><td>Login &amp; account</td></tr> <tr><td>When did this start?</td><td>this morning, ~08:30</td></tr> <tr><td>How many people are affected?</td><td>Just me</td></tr> </tbody> </table>

**Description**

> I typed my password wrong a couple of times this morning and now it says my account is locked. I waited a bit and tried again but it's still locked. I have a meeting at 10 and I can't get into anything.

## The Original Plan (7 Lab Steps)

Before starting the exercise, I had this 7-step plan laid out:

1. Reproduce the lockout on CLIENT01 (fail past the threshold on purpose)
2. Confirm the lockout state on DC01 with `Search-ADAccount -LockedOut`
3. Check the policy values (threshold / duration / observation window) with `Get-ADDefaultDomainPasswordPolicy`
4. Check Event 4740 (lockout) and 4625 (failed logon) in the DC01 Security log and pin down where it came from
5. Unlock with `Unlock-ADAccount`
6. Confirm the user can sign in again
7. Check whether it recurs (to tell human error apart from a cached credential issue)

In practice, a lot happened between those seven steps that I didn't see coming. Here's the whole thing, start to finish.

---

## Situation

This was a full solo run at an account lockout ticket, start to finish. My original plan was to just spin up a test account (tuser) and lock that instead. But basically the moment I started, I thought, "If I'm doing this, wouldn't it make more sense to use one of the company users I already set up?" A real helpdesk ticket would never come in from an account named tuser.

So I switched gears, and that one decision ended up making the prep work a lot longer than I expected.

## Symptoms

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-12.png?updatedAt=1788029215128)

The message on the CLIENT01 logon screen: `Your account has been locked out. Please contact your administrator or help desk.`

Honestly, getting locked out after 5 (or 3) failed attempts is something almost everyone runs into eventually, at work or anywhere else. The most familiar version of this for me is actually from a part-time job at Shoppers, where you had to log into the POS system with your own employee login. Staff had to change their password every six months, and forgetting it or mistyping it enough times to lock the account happened all the time. I've been there myself.

## Root Cause Investigation (Everything That Actually Happened)

### 1. Switching From a Test Account to a Real Company User

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-27.png?updatedAt=1788029215309)

I originally planned to create a new account called tuser and run the exercise on that. To make it feel closer to a real helpdesk scenario, I decided to use one of the company users I'd already built out instead. A real ticket comes in under a real employee's name, and using an existing account also brings along real variables like OU placement, group membership, and mapped drives.

### 2. Finalizing the Company Org Chart

Looking at a screenshot of the JSM customer list, I laid out the five people I'd set up so far.

```
LUMEN.LOCAL
├── IT
│   ├── John Smith (jsmith)
│   └── Emily Brown (ebrown)
├── HR
│   └── Sarah Kim (skim)
├── Sales
│   └── Michael Lee (mlee)
└── Finance
    └── David Wilson (dwilson)

```

### 3. Picking Who This Ticket Would Be About

I went with Michael Lee from Sales. The two IT people were risky to use since they might belong to admin groups, and dwilson already had a closed request logged, so I figured it was better to save him for a different ticket's history. Sales also fit the "locked out right before a client meeting" story well.

### 4. Getting Lost in the GUI While Checking If the Account Was Safe

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-01.png?updatedAt=1788029214693)

I needed to confirm Michael Lee's account wasn't part of any admin group like Domain Admins. I thought about doing this in PowerShell first, but I can't memorize every command, so I wanted to do it through the GUI instead.

I opened `dsa.msc` (Active Directory Users and Computers) and got as far as Michael Lee's properties window, but then I couldn't find "Member Of" anywhere and got a little stuck. For a minute I genuinely thought I just couldn't find it.

Once I did track it down, the only groups listed were `Domain Users` and `Sales-Users`. No Domain Admins, no Protected Users, nothing risky. Safe account to run the experiment on.

### 5. An Unexpected Find: Jessica Park

While I was in Michael Lee's properties window, I noticed another account in the same Sales OU that wasn't part of the plan at all: Jessica Park. Turned out I'd created it a while back and just forgotten about it. I thought about deleting or disabling it, but decided to leave it alone and save it for a future "coworker" ticket scenario instead.

### 6. Checking Account Options: Had to Turn Off "Must Change Password"

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-03.png?updatedAt=1788029214750)

The Account tab showed `User must change password at next logon` checked. Leaving that on would force a password change screen at logon, which would get in the way of building the profile and clutter up the event logs later (4723, 4738, that kind of thing), so I unchecked it. `Unlock account` was unchecked, which made sense since nothing was locked yet, and `Account expires` was set to Never, also fine.

### 7. Filling In Personal Details

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-02.png?updatedAt=1788029214714)

To make the "verify identity before unlocking" step actually real, I filled in the General and Organization tabs.

- Office: Toronto HQ - Floor 3
- Telephone number: 416-555-0142
- Job Title: Account Executive
- Department: Sales
- Company: Lumen Systems

### 8. Confirming the Password and Signing Into CLIENT01 for the First Time

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-04.png?updatedAt=1788029214978)

I still remembered the password I'd set when I created the account, so no reset needed. I signed into CLIENT01 once as `LUMEN\mlee` to build the profile ahead of time. Skip this step and the first logon throws a bunch of extra events into the mix, which makes the logs harder to read later.

### 9. Checking the Lockout Policy: The Values Didn't Match My Old Notes

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-07.png?updatedAt=1788029214919)

Running `Get-ADDefaultDomainPasswordPolicy` gave me `5 / 00:15:00 / 00:15:00`. That didn't match the values I'd written down from an earlier ticket (5 attempts / 30 minutes / 30 minutes), and for a second I panicked, thinking I'd misconfigured something. Turned out it was just the default value nobody had touched yet.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-08.png?updatedAt=1788029214905)

I decided to bump it to 30 minutes to stay consistent across my ticket write-ups. This is also when I learned `Get-ADDefaultDomainPasswordPolicy` is read-only, so actually changing the value meant going into GPMC (Group Policy Management) instead.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-09.png?updatedAt=1788029214923)

GUI path: `gpmc.msc` → right-click `Default Domain Policy` → Edit → Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies → Account Lockout Policy. I changed the duration and the reset counter to 30 minutes there, then ran `gpupdate /force` to push it through right away.

### 10. Reproducing the Lockout: Thrown Off by "Delaying Next Attempt"

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-11.png?updatedAt=1788029215119)

Before I even got to the actual lockout message, I ran into a completely different one first: `Delaying next attempt for security reasons.` Never seen it before, and it threw me for a while. Spoiler: it just meant wait it out, I scared myself over nothing.

I logged off and, at the logon screen, kept entering `LUMEN\mlee` with the wrong password on purpose. After a few tries, a message I'd never seen came up: `Delaying next attempt for security reasons.` For a second I genuinely panicked, thinking, "Wait, is this a 30-minute delay? Am I not going to finish this today? I have to leave for work soon..."

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-13.png?updatedAt=1788029214937)

I switched over to DC01 and ran `Search-ADAccount -LockedOut | Select-Object Name, SamAccountName, LockedOut`, and LockedOut came back True. So the account was already locked.

Turned out that message was just a local sign-in throttling feature on CLIENT01, completely separate from the domain's Account Lockout Policy. I waited out the delay shown on screen, tried again, and it locked normally on the fifth failure.

### 11. Confirming the Lockout on DC01

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-14.png?updatedAt=1788029214929)

In `dsa.msc`, Michael Lee's Account tab showed `Unlock account. This account is currently locked out on this Active Directory Domain Controller`, with the checkbox now enabled. `Search-ADAccount -LockedOut` confirmed the same thing in PowerShell. I hit Cancel to close the window without unlocking anything yet, since it was still investigation time, and moved on.

### 12. Digging Into Event Logs: Stuck for a While When 4771 Wouldn't Show Up

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-17.png?updatedAt=1788029214942)

I opened `eventvwr.msc`, filtered the Security log for 4740, and it showed up right away, `Caller Computer Name: CLIENT01` and all.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-18.png?updatedAt=1788029214959)

The problem was 4771 (Kerberos pre-authentication failure). I'd literally just typed the wrong password six times, but filtering for it turned up nothing. Where did it go...

Started wondering if I was doing something wrong, so I tried PowerShell too, and widened the net to query 4771, 4776, and 4740 all at once.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-15.png?updatedAt=1788029215375)

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4771,4776,4740; StartTime=(Get-Date).AddHours(-3)} |
  Select-Object TimeCreated, Id, Message | Format-List
```

The result was just one 4740 (6:57:55 AM). No 4771, no 4776. Still nothing. I asked the AI, "this command isn't working," wondering if it was because I hadn't opened PowerShell as admin, reopened it that way, and tried again. Still nothing.

### 13. The Real Cause: Auditing Was Turned Off

It occurred to me that maybe the logs just weren't being recorded at all, so I checked the audit policy with `auditpol`.

```powershell
auditpol /get /subcategory:"Kerberos Authentication Service"
```

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-31.png?updatedAt=1788029215366)

The result: `No Auditing`. This is when I learned that a default Windows Server install logs the lockout result (4740) but doesn't record individual failed attempts (4771) at all unless that audit subcategory is turned on. All that rerunning the command and second-guessing my admin rights, and the actual reason was that the logging was never turned on in the first place. If it says `Not Configured`, you just go turn it on.

The path to it was buried pretty deep. Definitely not easy to find.

**Domain GPO (`gpmc.msc` → edit the GPO)**
Computer Configuration → Policies → Windows Settings → Security Settings → Advanced Audit Policy Configuration → Audit Policies → Account Logon → **Audit Kerberos Authentication Service**

### 14. Turning On Auditing and Locking It Again to Get Evidence

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-22.png?updatedAt=1788029215349)

```powershell
auditpol /set /subcategory:"Kerberos Authentication Service" /success:enable /failure:enable
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
```

Once you turn this on, it doesn't apply retroactively, which meant... I had to start that part over. Ugh. Unlock the account, lock it again. Annoying, but not hard. I unlocked it with `Unlock-ADAccount` and reproduced the lockout on CLIENT01 one more time.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-23.png?updatedAt=1788029215337)

```
Kerberos pre-authentication failed.
Account Name: mlee
Service Name: krbtgt/LUMEN.LOCAL
Client Address: ::ffff:192.168.136.21
Client Port: 49998
```

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-24.png?updatedAt=1788029214675)

All seven came from the same Client Address (192.168.136.21, which is CLIENT01), Status `0x18` (bad password). I checked 4740 again too, this time `Caller Computer Name: CLIENT01` at `7:21:16 AM`. (The earlier 6:57:55 AM event was from before I turned auditing on, so that one's just for reference. I used 7:21:16 AM as the official record.)

Filtering Event Viewer for 4771 this time actually pulled up all seven.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-25.png?updatedAt=1788029215358)

Just to be extra sure, I opened `eventvwr.msc` again and filtered the Security log for 4740, the same one that hadn't shown its face earlier, and this time...

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-26.png?updatedAt=1788029214946)

There it was, finally. 4740 showed up in the filter. That's when I could actually prove, with the logs, that this really was someone mistyping a password at the logon screen. Evidence secured. Time to verify identity and wrap this up by unlocking the account.

### 15. Verifying Identity

Before touching the account again, I verified the caller's identity against what was on file in AD.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-39.png)

```powershell
Get-ADUser mlee -Properties Department, Title, Office, OfficePhone |
  Format-List Name, Department, Title, Office, OfficePhone
```

Department: Sales, Title: Account Executive, Office: Toronto HQ - Floor 3, OfficePhone: 416-555-0142. Because I'd filled that info in ahead of time, this step wasn't just something I typed for show, it had an actual paper trail behind it.

Honestly, you don't even need the formatting, just `Get-ADUser mlee` on its own already tells you who the user is and what they belong to. But identity's confirmed either way, so now I can go unlock the account.

## Resolution

### Unlocking the Account

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-33.png?updatedAt=1788029215354)

```powershell
Unlock-ADAccount -Identity mlee
Search-ADAccount -LockedOut   # empty result = unlock successful
```

`Search-ADAccount -LockedOut` came back empty, confirming the unlock worked.

If the command slips your mind, you can just as easily unlock it from the GUI.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-28.png?updatedAt=1788029215371)

Check the box next to `Unlock account. This account is currently locked out on this Active Directory Domain Controller`, hit Apply, then OK, and you're done.

### Confirming the Sign-In

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-34.png?updatedAt=1788029215331)

Signed into CLIENT01 as `LUMEN\mlee` and got the `Welcome` screen, logged in fine.

### The 20-Minute Recurrence Check

Felt like I could've stopped there, but I was told that anyone can unlock an account, and that checking whether it recurs is the part that actually separates a real Tier 1 from someone just clicking a button. So I ran the 20-minute monitoring loop. I really just wanted to close this ticket and move on, but apparently confirming it stays fixed is part of the job too.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-2/LUM2-37.png?updatedAt=1788029214742)

I sat there and genuinely waited the full 20 minutes.

```powershell
1..20 | ForEach-Object {
  $u = Get-ADUser mlee -Properties LockedOut, BadPwdCount, LastBadPasswordAttempt
  "{0:HH:mm:ss}  Locked={1}  BadPwdCount={2}  LastBad={3}" -f `
    (Get-Date), $u.LockedOut, $u.BadPwdCount, $u.LastBadPasswordAttempt
  Start-Sleep -Seconds 60
}
```

Along the way I also picked up the concept of a "stale cached credential." If a password gets changed but the old one is still sitting in Credential Manager or a mapped drive, it'll auto-retry every 30 minutes and re-lock the account. In this case, I'd never actually created a stored credential like that for mlee, so recurrence was unlikely, but I got why writing "no recurrence" without actually checking would just be a guess dressed up as verification.

From `07:39:02` to `07:58:02`, all 20 checks came back `Locked=False`, `BadPwdCount=0`. No recurrence. Confirmed one-off user error.

Honestly, I went in thinking, "It's just a lockout, you unlock it and move on, right?" I had no idea a ticket this simple would end up teaching me this much. What looked like a quick five-minute ticket turned out to have a lot more weight to it than I expected.

## Lessons Learned

- Don't go looking for GUI tabs in Korean. If the server's running the English UI, the tabs are in English too. (소속 그룹 = Member Of)
- When picking an account to experiment on, avoid Domain Admins, Enterprise Admins, Protected Users, and service accounts. One glance at the Member Of tab in the GUI tells you whether it's safe.
- Lockout policy defaults can vary by environment. When documenting multiple tickets, it's worth checking the values up front and keeping them consistent.
- `Get-ADDefaultDomainPasswordPolicy` is read-only. Actually changing a value means editing the Default Domain Policy itself through GPMC.
- The "Delaying next attempt" message is a separate, local feature on the client, unrelated to the domain's Account Lockout Policy. Even when it shows up, any failed attempt already sent gets recorded on the DC regardless.
- Audit logging isn't all on by default. 4740 (the result) gets logged automatically, but 4771/4625 (the individual attempts) need auditpol turned on separately. Before deciding there's no evidence just because the events aren't showing up, check auditpol first.
- Turning on an audit policy doesn't apply retroactively. If you need evidence, you have to turn the policy on first and then reproduce the incident.
- Even with 4740 alone, the `Caller Computer Name` field is enough to identify the source workstation. You can still do a baseline investigation without 4771.
- Unlocking an account is one line of code. The 20-minute recurrence check is what actually tells you whether you did the job right, and the key to that judgment call is telling LogonType 2 (a human typing it in) apart from LogonType 3 (something like a mapped drive auto-retrying).

---

> **The 20-minute check is the part that matters.** Anyone can unlock an account. Distinguishing a one-off typo from a stale credential that will re-lock the account every 30 minutes is the actual Tier 1 skill, and that last line shows you know the difference even though this case turned out simple.