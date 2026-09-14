---
title: "The Shortcut IT Support Uses Dozens of Times a Day: Windows + R"
description: "Instead of hunting through ten clicks of settings menus, press two keys and open it directly. Organized by the tickets you'll actually get."
pubDate: "Sep 14 2026"
heroImage: "https://ik.imagekit.io/stephanie/Blog-1/Blog01.png?updatedAt=1789312457735"
tags: ["Windows", "IT Support", "Help Desk", "Run Command", "Troubleshooting"]
---


## Eight keystrokes instead of ten clicks

You've connected remotely to a user's PC and you need to check their network adapter settings.

![Network and Internet page in the Windows Settings app](https://ik.imagekit.io/stephanie/Blog-1/Blog02.png?updatedAt=1789312458310)

The usual path looks like this: Start > Settings > Network & internet > Advanced network settings > More network adapter options. Five or six clicks. And the menu names and locations keep shifting depending on whether it's Windows 10 or 11, and which build.

![Adapter options reached through advanced network settings](https://ik.imagekit.io/stephanie/Blog-1/Blog03.png?updatedAt=1789312458305)

Press **Windows + R**, type `ncpa.cpl`, and that window opens in one step. It works identically on Windows 7 and Windows 11.

That's why IT support technicians never stop using the Run dialog.

---

## What is the Run dialog?

![The Run dialog opened with Windows + R](https://ik.imagekit.io/stephanie/Blog-1/Blog01.png?updatedAt=1789312457735)

It's the small input box you open with **Windows + R**. You type a program name or a management console filename and it launches directly.

Three reasons it earns its place on the job:

**1. It's fast.** Save thirty seconds per ticket across twenty tickets a day and you've saved ten minutes.

**2. It doesn't care which Windows version you're on.** The Settings app UI changes with every update, but `services.msc` has been the same for twenty years. You won't get lost on a user's machine no matter what build they're running.

**3. It looks competent during remote support.** There's a real difference in how a user perceives you when you open the right window instantly versus fumbling through menus on a shared screen. The same applies when you walk an interviewer through your troubleshooting process.

---

## The file extension rule worth knowing first

If the commands won't stick, learning the pattern helps.

<table>
  <thead>
    <tr>
      <th>Extension</th>
      <th>What it means</th>
      <th>Examples</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><strong>.msc</strong></td>
      <td>Microsoft Management Console. Admin consoles</td>
      <td><code>services.msc</code>, <code>eventvwr.msc</code></td>
    </tr>
    <tr>
      <td><strong>.cpl</strong></td>
      <td>Control Panel item</td>
      <td><code>ncpa.cpl</code>, <code>appwiz.cpl</code></td>
    </tr>
    <tr>
      <td><strong>.exe</strong> or just a name</td>
      <td>A regular executable</td>
      <td><code>cmd</code>, <code>regedit</code>, <code>mstsc</code></td>
    </tr>
  </tbody>
</table>

![The Services management console opened with services.msc](https://ik.imagekit.io/stephanie/Blog-1/Blog04.png?updatedAt=1789312458290)

`.msc` means an administrative tool.

![Programs and Features opened with appwiz.cpl](https://ik.imagekit.io/stephanie/Blog-1/Blog05.png?updatedAt=1789312458281)

`.cpl` means a Control Panel item. Remembering just those two gets you halfway there.

---

## Organized by the ticket

Memorizing an alphabetical list doesn't stick. Grouping them as "when this ticket comes in, I type this" does.

### Ticket: "The application suddenly won't open"

![Checking an error log in Event Viewer](https://ik.imagekit.io/stephanie/Blog-1/Blog06.png?updatedAt=1789312458430)

**`eventvwr.msc`** Event Viewer. Find when the error happened and what code it threw

![Checking service status in the Services console](https://ik.imagekit.io/stephanie/Blog-1/Blog04.png?updatedAt=1789312458290)

**`services.msc`** Check whether the relevant service stopped, then restart it

![The Processes tab in Task Manager](https://ik.imagekit.io/stephanie/Blog-1/Blog07.png?updatedAt=1789312458118)

**`taskmgr`** Task Manager. Look for a ghost process still hanging around

![Resource Monitor](https://ik.imagekit.io/stephanie/Blog-1/Blog08.png?updatedAt=1789312458222)

**`resmon`** Resource Monitor. Check for CPU, disk, or memory bottlenecks

Root cause work almost always starts in **Event Viewer**. When a user tells you "it just doesn't work," the log has the exact timestamp and error code.

### Ticket: "The internet isn't working"

![The Network Connections window opened with ncpa.cpl](https://ik.imagekit.io/stephanie/Blog-1/Blog09.png?updatedAt=1789312458082)

**`ncpa.cpl`** Network adapter list. Check whether an adapter is disabled

![Running ipconfig at the command prompt](https://ik.imagekit.io/stephanie/Blog-1/Blog10.png?updatedAt=1789312458517)

**`cmd`** Run ipconfig /all, ping, and nslookup

![The Network and Sharing Center](https://ik.imagekit.io/stephanie/Blog-1/Blog11.png?updatedAt=1789312458295)

**`control /name Microsoft.NetworkAndSharingCenter`** Network and Sharing Center

`ncpa.cpl` catches the anticlimactic "the adapter was simply switched off" in about ten seconds. It happens more often than you'd think.

### Ticket: "Please connect the printer on my new laptop"

![The Devices and Printers window](https://ik.imagekit.io/stephanie/Blog-1/Blog12.png?updatedAt=1789312458211)

**`control printers`** Devices and Printers

![Typing a server name into the Run dialog](https://ik.imagekit.io/stephanie/Blog-1/Blog13.png?updatedAt=1789312457996)

**`\\PRINTSRV01`** Connect to the company print server

Companies don't plug a printer into every desk with a cable. A single **print server** holds the printers, and staff pick the one they need from it.

Type the server name into Run and the list of printers it hosts opens up. Double-click the one you want and it installs. `PRINTSRV01` is just an example; the real name differs at every company, so check your internal documentation.

The point worth taking away here is that the Run dialog accepts more than program names. You can type a **network path in the form `\\servername`** directly. File servers work the same way. Just seeing whether `\\FS01` returns a list or not already narrows down the problem.

![The error saying printmanagement.msc cannot be found](https://ik.imagekit.io/stephanie/Blog-1/Blog14.png?updatedAt=1789312457787)

There's also a Print Management console, `printmanagement.msc`, but it often isn't installed by default on current Windows builds, so you'll get a "cannot be found" error. When that happens, `control printers` does the job.

### Ticket: "Can you uninstall / install this program?"

![The Programs and Features window](https://ik.imagekit.io/stephanie/Blog-1/Blog05.png?updatedAt=1789312458281)

**`appwiz.cpl`** Programs and Features

**`optionalfeatures`** Turn Windows features on or off (Telnet Client, .NET, and so on)

### Ticket: "My account is locked / please reset my password"

![A user account open in the ADUC console](https://ik.imagekit.io/stephanie/Blog-1/Blog17.png?updatedAt=1789312458227)

**`dsa.msc`** Active Directory Users and Computers (domain accounts)

![The Local Users and Groups console](https://ik.imagekit.io/stephanie/Blog-1/Blog18.png?updatedAt=1789312458149)

**`lusrmgr.msc`** Local Users and Groups (accounts that exist only on that PC)

![The netplwiz user accounts window](https://ik.imagekit.io/stephanie/Blog-1/Blog19.png?updatedAt=1789312458207)

**`netplwiz`** User account settings and automatic sign-in configuration

Domain accounts live in `dsa.msc`; accounts that exist only on that one machine live in `lusrmgr.msc`. Mix the two up and you'll spend the call trading "I don't see the account" back and forth.

Unlocking a domain account: Windows + R > dsa.msc → domain → Find → user → Properties → Account tab

### Ticket: "The policy change isn't taking effect"

![Running gpupdate /force at the command prompt](https://ik.imagekit.io/stephanie/Ticket-4/LUM4-21.png?updatedAt=1788029366229)

**`gpupdate /force`** Reapply policy immediately

![The Resultant Set of Policy console](https://ik.imagekit.io/stephanie/Blog-1/Blog20.png?updatedAt=1789312458178)

**`rsop.msc`** Check the resultant set of policy

![A GPO linked to an OU in the Group Policy Management Console](https://ik.imagekit.io/stephanie/Blog-1/Blog21.png?updatedAt=1789312458188)

**`gpmc.msc`** Group Policy Management Console (on a server, or a PC with RSAT installed)

![The Local Group Policy Editor](https://ik.imagekit.io/stephanie/Blog-1/Blog22.png?updatedAt=1789312458052)

**`gpedit.msc`** Local Group Policy Editor

`gpupdate /force` is a command prompt tool, but typing it straight into Run works too. The window flashes and disappears, so if you want to read the result, run it inside `cmd` instead.

### Ticket: "My PC is really slow"

![The System Configuration window](https://ik.imagekit.io/stephanie/Blog-1/Blog23.png?updatedAt=1789312458163)

**`msconfig`** System Configuration. Startup and boot options

![The Startup apps tab in Task Manager](https://ik.imagekit.io/stephanie/Blog-1/Blog24.png?updatedAt=1789312458363)

**`taskmgr`** Check the Startup apps tab

![The Disk Cleanup window](https://ik.imagekit.io/stephanie/Blog-1/Blog25.png?updatedAt=1789312458235)

**`cleanmgr`** Disk Cleanup

![The Optimize Drives window](https://ik.imagekit.io/stephanie/Blog-1/Blog26.png?updatedAt=1789312458389)

**`dfrgui`** Optimize Drives

![The temp folder open in File Explorer](https://ik.imagekit.io/stephanie/Blog-1/Blog27.png?updatedAt=1789312458244)

**`%temp%`** Open the temp folder directly to clear it out

As `%temp%` shows, **environment variables work in Run as well**. The ones you'll use most:

![The Roaming app data folder](https://ik.imagekit.io/stephanie/Blog-1/Blog28.png?updatedAt=1789312458405)

**`%appdata%`** Roaming application data

![The Local app data folder](https://ik.imagekit.io/stephanie/Blog-1/Blog29.png?updatedAt=1789312458527)

**`%localappdata%`** Local application data

![The Startup folder](https://ik.imagekit.io/stephanie/Blog-1/Blog30.png?updatedAt=1789312458155)

**`shell:startup`** The current user's Startup folder

`%appdata%` comes up in nearly every Outlook profile issue and app settings reset.

### Ticket: "The device isn't being detected"

![The Device Manager window](https://ik.imagekit.io/stephanie/Blog-1/Blog31.png?updatedAt=1789312458200)

**`devmgmt.msc`** Device Manager. Driver problems and yellow warning icons

![The Disk Management console](https://ik.imagekit.io/stephanie/Blog-1/Blog32.png?updatedAt=1789312458232)

**`diskmgmt.msc`** Disk Management. Initializing new disks, partitions, drive letters

![The Computer Management console](https://ik.imagekit.io/stephanie/Blog-1/Blog33.png?updatedAt=1789312458205)

**`compmgmt.msc`** Computer Management. All of the above in one window

`compmgmt.msc` alone gives you Device Manager, Disk Management, Event Viewer, Local Users, and Shared Folders in one place. One command to remember instead of five.

### Remote access and server work

![The Remote Desktop Connection window](https://ik.imagekit.io/stephanie/Blog-1/Blog34.png?updatedAt=1789312458253)

**`mstsc`** Remote Desktop Connection (use `mstsc /v:SERVER01` to connect to a specific server directly)

![The Computer Name tab in System Properties](https://ik.imagekit.io/stephanie/Blog-1/Blog35.png?updatedAt=1789312458162)

**`sysdm.cpl`** System Properties. Renaming the computer and joining a domain

![Windows version information shown by winver](https://ik.imagekit.io/stephanie/Blog-1/Blog36.png?updatedAt=1789312458373)

**`winver`** Windows version and build number

`sysdm.cpl` is the starting point for any domain join: Windows + R > sysdm.cpl → Computer Name tab → Change

`winver` looks trivial, but recording an exact build like "Windows 11 23H2" in the ticket makes the conversation far quicker when you **escalate** to a higher tier.

---

## One-page reference

<table>
  <thead>
    <tr>
      <th>Command</th>
      <th>What it opens</th>
      <th>When you'll use it</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>ncpa.cpl</code></td>
      <td>Network adapters</td>
      <td>Connectivity problems</td>
    </tr>
    <tr>
      <td><code>eventvwr.msc</code></td>
      <td>Event Viewer</td>
      <td>Root cause of errors</td>
    </tr>
    <tr>
      <td><code>services.msc</code></td>
      <td>Services</td>
      <td>Stopping and restarting services</td>
    </tr>
    <tr>
      <td><code>devmgmt.msc</code></td>
      <td>Device Manager</td>
      <td>Drivers and peripheral detection</td>
    </tr>
    <tr>
      <td><code>diskmgmt.msc</code></td>
      <td>Disk Management</td>
      <td>Partitions and drive letters</td>
    </tr>
    <tr>
      <td><code>compmgmt.msc</code></td>
      <td>Computer Management</td>
      <td>All of the above in one console</td>
    </tr>
    <tr>
      <td><code>dsa.msc</code></td>
      <td>AD Users and Computers</td>
      <td>Domain account work</td>
    </tr>
    <tr>
      <td><code>lusrmgr.msc</code></td>
      <td>Local Users and Groups</td>
      <td>Local account work</td>
    </tr>
    <tr>
      <td><code>gpmc.msc</code></td>
      <td>Group Policy Management</td>
      <td>Creating and linking GPOs</td>
    </tr>
    <tr>
      <td><code>gpedit.msc</code></td>
      <td>Local Group Policy</td>
      <td>Single-machine policy</td>
    </tr>
    <tr>
      <td><code>rsop.msc</code></td>
      <td>Resultant Set of Policy</td>
      <td>Confirming applied policy</td>
    </tr>
    <tr>
      <td><code>appwiz.cpl</code></td>
      <td>Programs and Features</td>
      <td>Uninstalling software</td>
    </tr>
    <tr>
      <td><code>sysdm.cpl</code></td>
      <td>System Properties</td>
      <td>Domain join, computer name</td>
    </tr>
    <tr>
      <td><code>mstsc</code></td>
      <td>Remote Desktop</td>
      <td>Connecting to servers and remote PCs</td>
    </tr>
    <tr>
      <td><code>msconfig</code></td>
      <td>System Configuration</td>
      <td>Boot and startup options</td>
    </tr>
    <tr>
      <td><code>taskmgr</code></td>
      <td>Task Manager</td>
      <td>Processes and performance</td>
    </tr>
    <tr>
      <td><code>regedit</code></td>
      <td>Registry Editor</td>
      <td>Advanced configuration changes</td>
    </tr>
    <tr>
      <td><code>cmd</code></td>
      <td>Command Prompt</td>
      <td>Network diagnostic commands</td>
    </tr>
    <tr>
      <td><code>%temp%</code></td>
      <td>Temp folder</td>
      <td>Disk cleanup</td>
    </tr>
    <tr>
      <td><code>%appdata%</code></td>
      <td>App data folder</td>
      <td>Profile and app settings issues</td>
    </tr>
    <tr>
      <td><code>control printers</code></td>
      <td>Devices and Printers</td>
      <td>Adding and removing printers</td>
    </tr>
    <tr>
      <td><code>winver</code></td>
      <td>Version information</td>
      <td>Recording the build in a ticket</td>
    </tr>
  </tbody>
</table>

---

## Two cautions

![The Registry Editor](https://ik.imagekit.io/stephanie/Blog-1/Blog37.png?updatedAt=1789312458171)

**Treat regedit carefully.** The registry is the Windows configuration database. Break the wrong thing and the machine may not boot. Always export the key before you change it, and record what you changed in the ticket. Applying a registry value you found online without understanding it is the single most common mistake juniors make.

**Some tools need elevation.** Tools like `services.msc` and `diskmgmt.msc` only let you make real changes when opened as administrator. Press **Ctrl + Shift + Enter** in the Run dialog to launch elevated.

---

## Closing

Memorizing Run commands isn't a skill in itself. What it changes is **how fast you get to the cause**. If you know where to look and you can open it in three seconds, the rest of your time goes into actual diagnosis.

You don't need to memorize the whole list. Start with the commands tied to the three or four ticket types you see most. I learned `eventvwr.msc`, `services.msc`, `ncpa.cpl`, and `dsa.msc` first.