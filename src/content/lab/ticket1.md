---
title: Ticket1 - Finance folder access denied.
description: "David Wilson`dwilson` from Finance reported that a shared folder he had been using normally until yesterday would no longer open this morning."
pubDate: 2026-08-23
heroImage: "https://ik.imagekit.io/stephanie/Ticket-1/LUM1-09.png?updatedAt=1788029183566"
badge: "IT Support"
tags:

  - Windows Server
  - File Server
  - NTFS Permissions 
---


## When the Share Opens but the Subfolder Denies You

**Ticket:** LUM-1 · Lumen Systems IT Service Desk  </br>
**Date:** 2026-08-23  </br>
**Category:** File Server / NTFS Permissions  </br>
**Environment:** Windows Server 2022 (DC01), Windows 11 (CLIENT01), domain `LUMEN.LOCAL`</br>

----------

## Situation

David Wilson (`dwilson`) from Finance reported that a shared folder he had been using normally until yesterday would no longer open this morning. The path was `\\DC01\CompanyData\Finance`, his own department's folder, so this is a location he was supposed to have access to in the first place.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-1/LUM1-26.png?updatedAt=1788029183249)

Because this was not a request for new access but a case of **something that used to work and stopped working**, it was handled as an incident rather than a service request. That distinction shapes the direction of everything that follows. The question is not "was he ever granted access" but "what changed."

----------

## Symptom

Reproduced on CLIENT01 while signed in as `dwilson`.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-1/LUM1-07.png?updatedAt=1788029183580)
```
\\DC01\CompanyData\Finance
```

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-1/LUM1-09.png?updatedAt=1788029183566)
```
You don't currently have permission to access this folder.
```

Three paths were checked back to back in the same session to narrow down the scope.

<table> <thead> <tr> <th>Path</th> <th>Result</th> <th>Interpretation</th> </tr> </thead> <tbody> <tr> <td><code>\\DC01\CompanyData</code></td> <td>Opens</td> <td>The share itself and the network connection are healthy</td> </tr> <tr> <td><code>\\DC01\CompanyData\Finance</code></td> <td>Denied</td> <td>Reported symptom successfully reproduced</td> </tr> <tr> <td><code>\\DC01\CompanyData\HR</code></td> <td>Denied</td> <td>Never had access to begin with (control case)</td> </tr> </tbody> </table>

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-1/LUM1-11.png?updatedAt=1788029183553)

It was important here not to mistake the HR denial for part of the problem. `dwilson` has never had access to the HR folder. Only **what worked yesterday and fails today** falls within the scope of this incident, and Finance is the only path that meets that condition.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-1/LUM1-12.png?updatedAt=1788029183541)
The fact that the parent folder `CompanyData` opens was another significant clue. If the share connection itself were broken, the parent folder would not have opened either. That means the fault sits somewhere below the share layer, not at it.

----------

## Investigation

### First hypothesis: group membership (it wasn't)

The most common cause was ruled out first. Most department folder access problems come down to the user having fallen out of the relevant group.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-1/LUM1-13.png?updatedAt=1788029183320)

On DC01:

```powershell
Get-ADGroupMember Finance-Users | Select name
```

`David Wilson` came back normally, so that side is fine.
The actual token on the client side was checked as well.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-1/LUM1-14.png?updatedAt=1788029183771)
```
whoami /groups | findstr /i Finance
```

Membership was intact here too. The first hypothesis was wrong, and this is where the real investigation begins.

### Dead end: looking for a share that doesn't exist

The next step was to inspect share permissions, so a per-department share was queried.

```powershell
Get-SmbShareAccess -Name Finance
```

```
Get-SmbShareAccess : No MSFT_SMBShare objects found with property 'Name' equal to 'Finance'.
```
![enter image description here](https://ik.imagekit.io/stephanie/Ticket-1/LUM1-15.png?updatedAt=1788029183362)
I had **assumed** the shares were split up by department, but the actual structure was different. The right move would have been to check which shares actually exist first.

```powershell
Get-SmbShare | Select Name, Path
```

There is only one share, `CompanyData`, and the department folders are nothing more than subfolders inside it. This failure is what gave me an accurate picture of the structure. One command taught me not to guess at an environment but to query it and confirm.

### Second check: share permissions (blocking nothing)

```powershell
Get-SmbShareAccess -Name CompanyData
```

```
Name        ScopeName AccountName AccessControlType AccessRight
----        --------- ----------- ----------------- -----------
CompanyData *         Everyone    Allow             Full

```

Share permissions grant `Everyone` `Full Control`. This layer blocks **no one, for any folder.**

And that result supplies the decisive piece of reasoning. A single share covers the entire tree, so share permissions **cannot explain a result that differs from folder to folder** — `CompanyData` opening while `Finance` is denied. The cause has to be at the NTFS layer.

Had I answered "permissions look fine" here and closed the ticket, it would have been a clear misdiagnosis. A share layer that looks healthy proves nothing.

### Third check: NTFS permissions (root cause found)
![enter image description here](https://ik.imagekit.io/stephanie/Ticket-1/LUM1-16.png?updatedAt=1788029183343)

```powershell
icacls C:\CompanyData\Finance
```

```
C:\CompanyData\Finance BUILTIN\Administrators:(F)
```

The `Finance-Users` entry is gone. On its own, though, this output is hard to judge as abnormal or not. Nothing errors out in a log; this is the kind of fault where **an entry that should be there is silently missing.**

So I compared it against folders of the same kind that were working normally.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-1/LUM1-17.png?updatedAt=1788029183326)

```powershell
icacls C:\CompanyData\Finance
icacls C:\CompanyData\HR
icacls C:\CompanyData\Sales
```
![enter image description here](https://ik.imagekit.io/stephanie/Ticket-1/LUM1-18.png?updatedAt=1788029183283)
HR and Sales each have their department group registered with `Modify`, while Finance had nothing but the administrators entry. Placed side by side, the anomaly was immediately obvious.

### Confirming before acting: Effective Access

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-1/LUM1-19.png?updatedAt=1788029183314)

I verified the diagnosis before fixing anything. Folder Properties → Security → Advanced → **Effective Access** tab, selecting `dwilson`, showed both read and write denied.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-1/LUM1-20.png?updatedAt=1788029183550)

Fixing on a hunch and calling it done is not the same as securing evidence before you act. This screen is what lets you answer "why did you conclude that?" later on.

----------

## Resolution

The `Finance-Users` group was restored to the Finance folder's NTFS ACL with `Modify`.

![enter image description here](https://ik.imagekit.io/stephanie/Ticket-1/LUM1-22.png?updatedAt=1788029183243)

**PowerShell**

```powershell
icacls C:\CompanyData\Finance /grant "LUMEN\Finance-Users:(OI)(CI)M"
```
![enter image description here](https://ik.imagekit.io/stephanie/Ticket-1/LUM1-21.png?updatedAt=1788029183238)
<table> <thead> <tr> <th>Component</th> <th>Meaning</th> </tr> </thead> <tbody> <tr> <td><code>(OI)</code></td> <td>Object Inherit &mdash; inherits to child <strong>files</strong></td> </tr> <tr> <td><code>(CI)</code></td> <td>Container Inherit &mdash; inherits to child <strong>folders</strong></td> </tr> <tr> <td><code>M</code></td> <td>Modify &mdash; read, write, change, delete</td> </tr> </tbody> </table>

**GUI**

1.  Right-click `C:\CompanyData\Finance` → Properties → Security tab
2.  Edit → Add
3.  Enter `Finance-Users` → Check Names
4.  Tick **Modify** under Allow
5.  Apply → OK

Under the Advanced tab, I confirmed that Applies to was set to `This folder, subfolders and files`. That is the GUI equivalent of `(OI)(CI)`.

**Verification**
![enter image description here](https://ik.imagekit.io/stephanie/Ticket-1/LUM1-23.png?updatedAt=1788029183349)

1.  Effective Access re-run: both read and write switched to granted
2.  Opened the folder as `dwilson`, created a test file and deleted it successfully
3.  **Least privilege check**: confirmed HR and Sales are still denied

Step 3 is not optional. Opening up more access than necessary while fixing an issue happens often in practice, and that is a bigger problem than the original one.

> Note: if an SMB session is already open on the client, a permission change may not take effect immediately. Run `net use * /delete /y` and reconnect, or sign out and back in, to be certain.

----------

## Takeaways

**The order I'll check in the next time I see this symptom.**

1.  **Narrow the scope first.** Does the parent folder open? Do other users see the same thing? What about other folders? You have to know how far normal extends before you can tell where abnormal begins.
2.  **Look only at the delta from the working state.** An incident is not "no access," it is "worked yesterday, fails today." A folder that was always closed is a control case, not a problem.
3.  **Rule out group membership first.** It is the most common cause, and when it turns out not to be, that is where things really start.
4.  **Inspect share permissions and NTFS permissions separately.** This is the core of this ticket.
5.  **Compare against a peer that works normally.** The fastest approach when you don't yet know what you're looking for.
6.  **Confirm the diagnosis with Effective Access before fixing anything.**

### Key concept: share permissions and NTFS permissions are separate layers

The two layers are evaluated independently, and what the user actually gets is **the more restrictive of the two.**

<table> <thead> <tr> <th>Layer</th> <th>This lab's configuration</th> <th>Actual role</th> </tr> </thead> <tbody> <tr> <td>Share</td> <td>Everyone / Full Control</td> <td>Controls nothing</td> </tr> <tr> <td>NTFS</td> <td>Department group per folder</td> <td>All real access control</td> </tr> </tbody> </table>

Leaving the share open and enforcing everything at NTFS is a widely used convention. Maintaining restrictions in two places makes effective permissions hard to trace.

In this kind of setup, checking only share permissions and reporting "everything looks normal" is a misdiagnosis every single time. **A healthy share permission proves nothing on its own.**

### Follow-up recommendations

-   Changing `Everyone` to `Authenticated Users` on the share would additionally rule out the possibility of anonymous access.
-   An ACE does not remove itself. It's worth reviewing who holds change rights over the `CompanyData` tree, and whether that scope should be narrowed.