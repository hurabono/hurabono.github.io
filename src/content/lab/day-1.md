---
title: "Building an Active Directory Home Lab — Day 1: From Environment Setup to Domain Controller Promotion"
description: "As part of preparing my portfolio for IT Support roles, I decided to build out the infrastructure for a fictional company, Lumen Systems, from the ground up. The first step was tackling one of the systems IT Support techs run into constantly in the real world: Active Directory. Today's goal was to spin up Windows Server 2025 on top of VMware Workstation and promote it all the way to a Domain Controller."
pubDate: 2026-08-05
badge: "IT Support"
tags:
  - IT Support
  - Windows Server
  - Home Lab
---

## Building an Active Directory Home Lab — Day 1: From Environment Setup to Domain Controller Promotion

As part of preparing my portfolio for IT Support roles, I decided to build out the infrastructure for a fictional company, **Lumen Systems**, from the ground up. The first step was tackling one of the systems IT Support techs run into constantly in the real world: **Active Directory (AD)**. Today's goal was to spin up Windows Server 2025 on top of VMware Workstation and promote it all the way to a Domain Controller (DC).

In an IT Support role, you eventually run into the systems that answer two basic questions: "who's allowed to log in?" and "which computers belong to this network?" Active Directory is the backbone behind both of those answers. There's a real difference between reading about it and actually building it with your own hands — and I felt that difference clearly today.

## Today's Objectives

- [x] Install VMware Workstation
- [x] Install Windows Server 2025
- [x] Install VMware Tools
- [x] Rename the server (DC01)
- [x] Configure a static IP
- [x] Install the Active Directory Domain Services (AD DS) role
- [x] Create a new forest: `lumen.local`
- [x] Promote the server to a Domain Controller

## Lab Environment Summary

| Component | Value |
|---|---|
| Hypervisor | VMware Workstation |
| Server OS | Windows Server 2025 |
| Domain | lumen.local |
| Server Name | DC01 |

## Reference Downloads

- VMware Workstation https://www.vmware.com/products/desktop-hypervisor/workstation-and-fusion
- Broadcom Download Portal (login required)  https://support.broadcom.com/group/ecx/downloads
- Windows Server 2025 (Evaluation) https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2025


---

## Step 1 — Install VMware Workstation

![VMware Workstation installer](https://ik.imagekit.io/stephanie/IT%20support%20Day1/1.png)

![VMware Workstation installation complete](https://ik.imagekit.io/stephanie/IT%20support%20Day1/1-1.png)

First things first: get the virtualization environment ready. VMware Workstation is a **Type-2 hypervisor**. Unlike Type-1 hypervisors (e.g., ESXi or Hyper-V's server mode), which run directly on bare-metal hardware, a Type-2 hypervisor runs as an application on top of an existing host OS — Windows, in my case.

That structure means I don't need a rack of physical servers to spin up a "server." I can run several virtual machines (VMs) on a single laptop, and each VM gets its own virtual CPU, RAM, storage, and network adapter — behaving as if it were an independent machine, even though it's really just one process among several on the same physical box. That's the biggest advantage of a home lab: it lets you recreate a domain environment without needing an actual company lab.

## Step 2 — Install Windows Server 2025

![VMware Installation](https://ik.imagekit.io/stephanie/IT%20support%20Day1/2.png)

VMware offers an "Easy Install" option that lets the wizard automatically configure CPU, memory, disk, and network settings. I deliberately skipped it and configured everything manually instead:

- CPU allocation
- Memory (RAM)
- Disk size
- Network adapter type

The reasoning was simple: letting the wizard decide everything means you never really learn *why* those settings matter. Setting each value by hand gave me a much better feel for how virtual hardware actually gets allocated to a VM, instead of treating it as a black box.

**Sizing note:** For a single Domain Controller lab VM running on a laptop, **2 vCPUs and 4 GB of RAM** turned out to be the sweet spot — enough for AD DS, DNS, and general Server Manager work to feel responsive, without starving the host machine of resources while other VMs or apps are running alongside it.

![VM hardware configuration — 2 vCPU / 4GB RAM](https://ik.imagekit.io/stephanie/IT%20support%20Day1/3.png)

**Boot Manager note:** The first time the VM boots, a **Boot Manager** screen pops up asking which device to boot from. Don't panic if you see this — it's expected. Just select **VMware Virtual SATA CD-ROM Drive** and continue, since that's where the Windows Server ISO is mounted.

![Boot Manager — selecting VMware Virtual SATA](https://ik.imagekit.io/stephanie/IT%20support%20Day1/3-2.png)

**Image selection note:** When the Windows Server installer asks which image to install, go with **Windows Server 2025 Standard (Desktop Experience)** rather than the Server Core option. Desktop Experience gives you the full GUI, Server Manager, and MMC snap-ins — much easier to work with while you're still learning AD DS, DNS, and Group Policy hands-on. Server Core is great for production once you're comfortable, but it's not the friendliest starting point for a first home lab.

![Selecting the Windows Server image — Desktop Experience](https://ik.imagekit.io/stephanie/IT%20support%20Day1/3-3.png)

**Troubleshooting note:** I almost downloaded an unnecessary language pack during setup that wasn't needed and could have caused issues later. Caught it before it installed. It was a good reminder that even a small, seemingly harmless option during setup can end up adding unnecessary overhead to storage or future updates.


## Step 3 — Install VMware Tools

![VMware Tools installation](https://ik.imagekit.io/stephanie/IT%20support%20Day1/4.png)

Right after creating a VM, things like screen resolution and mouse movement between the host and guest can feel clunky. Installing VMware Tools fixes most of that.

**Why install VMware Tools?**
- Better graphics performance
- Seamless mouse integration between host and guest
- Shared clipboard
- Improved network drivers
- Time synchronization with the host

The difference is easy to see side by side. Before installing VMware Tools, the display was stuck at a low, non-native resolution and felt sluggish to interact with:

![Display before installing VMware Tools — low resolution, no mouse integration](https://ik.imagekit.io/stephanie/IT%20support%20Day1/4-2.png)

After installing VMware Tools, the resolution snapped to a proper size and the mouse moved seamlessly between host and guest:

![Display after installing VMware Tools — proper resolution and smooth mouse integration](https://ik.imagekit.io/stephanie/IT%20support%20Day1/4-1.png)

That last point — time sync — turns out to matter a lot in an AD environment down the line. Active Directory authentication (Kerberos) is time-sensitive, so if the clocks on the server and clients drift apart, logins can start failing outright.

## Step 4 — Rename the Server

![VMware Installation](https://ik.imagekit.io/stephanie/IT%20support%20Day1/5.png)

The server initially had an auto-generated name like `WIN-Q60M6TI1AKG`. Before promoting it to a Domain Controller, I renamed it to `DC01` to follow standard enterprise naming conventions.

```
DC01  → Domain Controller
FS01  → File Server
SQL01 → SQL Server
WEB01 → Web Server
```

**Why rename the server?**

Meaningful server names make administration far easier. In an environment with dozens or hundreds of servers, being able to tell what a server does just from its name can make a big difference in how quickly an issue gets diagnosed and resolved. As an IT Support tech, being able to figure out "which server this ticket is even talking about" quickly is genuinely important — and this naming convention is the first step toward that.

Rebooted after the rename to apply the change.

## Step 5 — Configure a Static IP

![Network adapter properties](https://ik.imagekit.io/stephanie/IT%20support%20Day1/5.png)

![Configuring static IPv4 settings](https://ik.imagekit.io/stephanie/IT%20support%20Day1/5-1.png)

![Verifying the static IP with ipconfig](https://ik.imagekit.io/stephanie/IT%20support%20Day1/5-2.png)

**IP configuration** (verified via `ipconfig`):

```
IP Address:
Subnet Mask:
Default Gateway:
Preferred DNS:
```

**Why does a Domain Controller need a static IP?**

A regular computer usually requests an IP address from a DHCP server, and that address can change from day to day — `192.168.1.50` today, `192.168.1.77` tomorrow.

But a Domain Controller is the "reference address" every computer on the network looks up to authenticate and locate domain resources. If that address keeps changing, client PCs lose track of where the Domain Controller actually is — similar to how a delivery driver can't find your house if your street address changes every day.

So the takeaway is simple: **a static IP is an address that never changes**, and that's essential for keeping a Domain Controller reliably reachable.

## Step 6 — Install Active Directory Domain Services (AD DS)

![Adding the AD DS role in Server Manager](https://ik.imagekit.io/stephanie/IT%20support%20Day1/6-1.png)

Now for the core piece: installing the AD DS role on the server.

**What is AD DS?**

Active Directory Domain Services provides centralized authentication and management for users, computers, and groups across a network. The convenience of logging into multiple computers and resources with a single company account exists because of this service.

Right after the role finishes installing — before any promotion has happened — the Active Directory tools are notably **absent** from the Tools menu in Server Manager:

![Tools menu right after AD DS role install — no Active Directory tools yet](https://ik.imagekit.io/stephanie/IT%20support%20Day1/6.png)

![AD DS role installation progress and results](https://ik.imagekit.io/stephanie/IT%20support%20Day1/6-2.png)

## Step 7 — Create the Forest

![VMware Installation](https://ik.imagekit.io/stephanie/IT%20support%20Day1/7.png)

I named the forest `lumen.local`.

**Why create a forest?**

A forest is the highest-level logical container in Active Directory's structure. Domains, trees, and organizational units (OUs) all live inside it. In simple terms, this was the first outer boundary drawn around the entire IT org chart.

## Step 8 — Promote to Domain Controller

![VMware Installation](https://ik.imagekit.io/stephanie/IT%20support%20Day1/8.png)

After running the promotion wizard to completion, the server was successfully promoted to a Domain Controller.

One thing I found interesting: the Active Directory management tools **don't appear in Server Manager right after installing the AD DS role**. They only show up under the Tools menu once the server has actually completed promotion to a Domain Controller and rebooted. At first I thought something had gone wrong with the install, but once I realized the promotion process and the tools' visibility are separate steps, it made sense.

Final state after reboot:
- **Server Name:** DC01
- **Domain:** lumen.local

---

## What I Learned Today

Summing up the day's work:

- What VMware Workstation is and how virtual machines actually work
- How to install Windows Server 2025 with manual hardware configuration
- How VMware Tools affects VM performance and usability
- Why meaningful server naming conventions matter in real environments
- Why Domain Controllers require a static IP address
- How to install Active Directory Domain Services
- How to create a new Active Directory forest
- How to promote a server to a Domain Controller

Looking back, today's work boils down to "install, rename, set a static IP, add one role." But walking through the *why* behind each step made it feel less like a series of clicks and more like following the design principles behind a real enterprise environment.

## Next Steps

- [ ] Set up DNS Manager
- [ ] Work with Active Directory Users and Computers (ADUC)
- [ ] Create Organizational Units (OUs)
- [ ] Create user accounts
- [ ] Join a Windows 11 client to the domain