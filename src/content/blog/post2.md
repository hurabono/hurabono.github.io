---
title: "IT Support Terminology Reference"
description: "The terms a junior IT support technician actually hears in their first week, with not just the definitions but how each one shows up in real tickets.There are a lot of abbreviations, which can be confusing at first, but once you understand them, they can be very helpful and useful in IT Support."
pubDate: "Sep 15 2026"
heroImage: "https://ik.imagekit.io/stephanie/Blog-2/blog2-4.png?updatedAt=1789504978435"
tags: ["Active Directory", "NTFS", "Group Policy", "Help Desk"]
---


## IT Support Terminology Reference (Windows / Active Directory Environments)

The terms a junior IT support technician actually hears in their first week, with not just the definitions but how each one shows up in real tickets.

1. Active Directory
2. File permissions
3. Networking fundamentals
4. Endpoint management
5. Accounts and security
6. Microsoft 365
7. Service desk process terminology

---

## 1. Active Directory

### AD (Active Directory) / AD DS

Microsoft's directory service. A central database holding the organization's user accounts, computers, printers, and groups. Its proper name is **AD DS (Active Directory Domain Services)**.

![Active Directory Domain Services running on a domain controller](https://ik.imagekit.io/stephanie/Blog-2/blog2-1.png?updatedAt=1789504978283)

- Structure: **Forest > Domain > OU > Object**
- Uses **Kerberos** for authentication by default (NTLM for legacy compatibility)
- Queried over **LDAP** (port 389, or 636 for LDAPS)
- AD is **completely dependent on DNS**. If DNS fails, sign-ins, Group Policy, and domain joins fail with it.

> On the floor: "She can't log in on the new laptop." Usually an AD account issue, or a DNS / domain join issue.

### DC (Domain Controller)

![The domain controller in Server Manager](https://ik.imagekit.io/stephanie/Blog-2/blog2-2.png?updatedAt=1789504978457)

The server that holds the AD database (NTDS.dit) and processes authentication requests. Organizations normally run two or more and keep them in sync through **replication**. With only one, every employee loses the ability to log in the moment it goes down.

### OU (Organizational Unit)

An **organizational container** that holds objects inside a domain, usually structured by department or location. (e.g. `OU=Finance`, `OU=IT`, `OU=Toronto-Office`)

![Departmental OUs expanded under the domain in ADUC](https://ik.imagekit.io/stephanie/Blog-2/blog2-3.png?updatedAt=1789504978418)

As the screenshot shows, the domain is divided into OUs by department.

OUs matter for two reasons.

1. They are **the only container you can link a GPO to**. (See below for what a GPO is.)
2. They support **delegation**. You can grant the help desk permission to reset passwords for accounts inside the Sales OU and nothing else.

Watch out: the default `Users` and `Computers` objects are **containers (CN), not OUs**. You cannot link a GPO to them. That's why the final step of creating an account is always moving it into the correct OU.

### GPO (Group Policy Object)

![A Group Policy Object open for editing](https://ik.imagekit.io/stephanie/Blog-2/blog2-9.png?updatedAt=1789504978429)

A bundle of settings **pushed out to users and computers automatically**. Password rules, desktop configuration, mapped drives, USB blocking, screen lock timeouts, and software deployment all live here.

- **Computer Configuration**: applied at boot, affects everyone who uses that machine
- **User Configuration**: applied at sign-in, follows that person to any machine

![Group Policy Objects linked at the domain and OU levels](https://ik.imagekit.io/stephanie/Blog-2/blog2-5.png?updatedAt=1789504978409)

**Processing order is LSDOU**: Local → Site → Domain → OU, and **whatever applies last wins.** An OU-level policy overrides a domain-level one.

![The order in which linked policies are processed](https://ik.imagekit.io/stephanie/Blog-2/blog2-6.png?updatedAt=1789504978389)

The exceptions are `Enforced` and `Block Inheritance`.

![The Enforced and Block Inheritance options in the Group Policy Management Console](https://ik.imagekit.io/stephanie/Blog-2/blog2-7.png?updatedAt=1789504978426)

![Group Policy results shown at the command line](https://ik.imagekit.io/stephanie/Blog-2/blog2-8.png?updatedAt=1789504978479)

Commands you'll use constantly:

<table>
  <thead>
    <tr>
      <th>Command</th>
      <th>Purpose</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>gpupdate /force</code></td>
      <td>Reapply policy immediately</td>
    </tr>
    <tr>
      <td><code>gpresult /r</code></td>
      <td>Show policies applied to this user and computer</td>
    </tr>
    <tr>
      <td><code>gpresult /h report.html</code></td>
      <td>Export a readable HTML report</td>
    </tr>
    <tr>
      <td><code>rsop.msc</code></td>
      <td>View resultant set of policy in a GUI</td>
    </tr>
  </tbody>
</table>

> Recurring ticket: "I changed the policy but nothing happened." Either the 90-minute refresh hasn't run, the machine wasn't rebooted or the user didn't sign in again, or the object isn't inside the OU the policy is linked to.

### GPMC / ADUC / RSAT

![The Active Directory Users and Computers console](https://ik.imagekit.io/stephanie/Blog-2/blog2-3.png?updatedAt=1789504978418)

- **ADUC (Active Directory Users and Computers)**: the console for creating accounts, resetting passwords, and managing group membership. `dsa.msc`

![The Group Policy Management Console](https://ik.imagekit.io/stephanie/Blog-2/blog2-4.png?updatedAt=1789504978435)

- **GPMC (Group Policy Management Console)**: where you create and link GPOs. `gpmc.msc`

![Installing Remote Server Administration Tools on a Windows client](https://ik.imagekit.io/stephanie/Blog-2/blog2-10.png?updatedAt=1789504978248)

- **RSAT (Remote Server Administration Tools)**: the feature pack that lets you run the tools above from your own Windows 10/11 machine instead of logging onto the server

### Security Group vs Distribution Group

![Group type options in Active Directory](https://ik.imagekit.io/stephanie/Blog-2/blog2-11.png?updatedAt=1789504978404)

- **Security Group**: used to grant permissions. Folder access, application licensing, VPN access.
- **Distribution Group**: used for email distribution only. Cannot grant permissions.

Standard practice: assign permissions **to groups, never to individual users**. When people change roles you add or remove them from a group instead of rebuilding permissions.

### Account terminology

![A user account's properties in ADUC](https://ik.imagekit.io/stephanie/Blog-2/blog2-12.png?updatedAt=1789504978413)

- **UPN (User Principal Name)**: a sign-in ID in the form `jane.doe@company.com`
- **SAMAccountName**: the legacy sign-in format, `COMPANY\jdoe`
- **Account lockout**: the account is locked after too many failed password attempts. Unlocking and resetting are two different actions.
- **Disable vs Delete**: departing employees are normally **disabled, not deleted**, so data access and audit history are preserved.

### Entra ID (formerly Azure AD)

The cloud version of the directory. The most common setup today is **hybrid**, syncing on-premises AD to the cloud with **Entra Connect**. This is where AZ-900 material connects directly to the job.

---

## 2. File Permissions

### NTFS (New Technology File System)

![A volume formatted as NTFS](https://ik.imagekit.io/stephanie/Blog-2/blog2-15.png?updatedAt=1789504978443)

The default Windows file system. When an IT support technician talks about NTFS, they're almost always talking about **permissions**.

Key features: per-file permissions (ACLs), journaling for recoverability, file compression and encryption (EFS), support for large volumes.

<table>
  <thead>
    <tr>
      <th>File system</th>
      <th>Permissions</th>
      <th>Typical use</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>NTFS</td>
      <td>Yes</td>
      <td>Internal Windows disks, file servers</td>
    </tr>
    <tr>
      <td>FAT32</td>
      <td>No</td>
      <td>Older USB drives, 4GB single-file limit</td>
    </tr>
    <tr>
      <td>exFAT</td>
      <td>No</td>
      <td>Large USB drives, Mac and Windows interchange</td>
    </tr>
  </tbody>
</table>

### NTFS permission levels

![NTFS permissions listed on the Security tab of a folder's properties](https://ik.imagekit.io/stephanie/Blog-2/blog2-11.png?updatedAt=1789504978404)

Right-click a folder > Properties > Security tab and you'll find this list.

Full Control / Modify / Read & Execute / List Folder Contents / Read / Write

![A Deny entry set on a folder's permissions](https://ik.imagekit.io/stephanie/Blog-2/blog2-21.png?updatedAt=1789504978396)

There is also **Deny**, and **Deny always beats Allow.** It's the first thing to check when troubleshooting an access problem.

### ACL / ACE

- **ACL (Access Control List)**: the list of who can do what on a given file or folder
- **ACE (Access Control Entry)**: a single line in that list

### Inheritance

Permissions flowing automatically from a parent folder down to its children. Break inheritance on a subfolder and it becomes independently managed from that point on. Most permission messes come from someone breaking inheritance and forgetting about it.

### Share permissions vs NTFS permissions

When a user connects over the network, **both sets apply**, and **the more restrictive one wins**.

![Share permissions on a shared folder](https://ik.imagekit.io/stephanie/Blog-2/blog2-16.png?updatedAt=1789504978273)

Example: Share = Full Control, NTFS = Read, actual result = **Read**

![Share permissions set to Authenticated Users with Full Control](https://ik.imagekit.io/stephanie/Blog-2/blog2-13.png?updatedAt=1789504978381)

Common practice: leave share permissions open at `Authenticated Users - Full Control` and **do the real control in NTFS**.

### Related terms

- **UNC path**: a network path in the form `\\FS01\Finance`
- **Mapped drive**: that UNC path attached to a drive letter such as Z:, usually deployed through GPO
- **Effective Access**: the tab that calculates what a specific user can actually do. The key tool for resolving permission tickets.
- **Take Ownership**: claiming ownership of an object. Used when nobody can access a departed employee's folder.
- **VSS / Shadow Copy**: restores previous versions for the "I deleted a file by accident" ticket

---

## 3. Networking Fundamentals

<table>
  <thead>
    <tr>
      <th>Term</th>
      <th>What it is</th>
      <th>Where support sees it</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><strong>DHCP</strong></td>
      <td>Hands out IP addresses automatically</td>
      <td>No lease means a 169.254.x.x (APIPA) address</td>
    </tr>
    <tr>
      <td><strong>DNS</strong></td>
      <td>Resolves names to IP addresses</td>
      <td>"The internet works but our internal systems don't"</td>
    </tr>
    <tr>
      <td><strong>Default Gateway</strong></td>
      <td>Exit point toward external networks</td>
      <td>Internal resources work, external ones don't</td>
    </tr>
    <tr>
      <td><strong>Subnet mask</strong></td>
      <td>Defines the local network range</td>
      <td>Wrong value breaks traffic even between neighbouring PCs</td>
    </tr>
    <tr>
      <td><strong>VPN</strong></td>
      <td>Secure remote access to the corporate network</td>
      <td>The number one remote-work ticket</td>
    </tr>
    <tr>
      <td><strong>VLAN</strong></td>
      <td>Logical segmentation of a physical switch</td>
      <td>Separating guest and corporate networks</td>
    </tr>
    <tr>
      <td><strong>NAT</strong></td>
      <td>Translates private IPs to public ones</td>
      <td>How a home or office router works</td>
    </tr>
    <tr>
      <td><strong>Proxy</strong></td>
      <td>Relays and filters web traffic</td>
      <td>"Only this one site won't load"</td>
    </tr>
    <tr>
      <td><strong>Port</strong></td>
      <td>Numeric identifier for a service</td>
      <td>RDP 3389, HTTPS 443, SMB 445</td>
    </tr>
    <tr>
      <td><strong>RDP</strong></td>
      <td>Remote Desktop Protocol</td>
      <td>Server access and remote support</td>
    </tr>
    <tr>
      <td><strong>MAC address</strong></td>
      <td>Physical address of a network adapter</td>
      <td>IP reservations, network registration</td>
    </tr>
  </tbody>
</table>

### Essential commands

![Output of ipconfig /all at the command prompt](https://ik.imagekit.io/stephanie/Blog-2/blog2-17.png?updatedAt=1789504978573)

Most of the terms in the table above show up as real values on this single screen. IP address, subnet mask, default gateway, DNS server, and MAC address are all right there.

```
ipconfig /all          Current IP, DNS, and DHCP server
ipconfig /release      Release the current lease
ipconfig /renew        Request a new lease
ipconfig /flushdns     Clear the DNS cache
ping <target>          Test connectivity
nslookup <domain>      Test name resolution
tracert <target>       Trace where the path breaks
netstat -ano           Show ports and owning processes
```

Troubleshoot **from the bottom up**: cable or Wi-Fi → IP address → ping the gateway → DNS → the application itself.

---

## 4. Endpoint Management

- **Imaging**: applying a standard Windows image to a new machine. Core onboarding work.
- **Intune / SCCM (MECM)**: tools for deploying apps and updates to corporate machines remotely
- **MDM (Mobile Device Management)**: managing phones and tablets, including remote wipe when a device is lost
- **BIOS / UEFI**: firmware. Boot order, Secure Boot settings.
- **TPM**: the security chip that stores encryption keys. A prerequisite for BitLocker.
- **BitLocker**: full disk encryption. The critical part is backing up the **recovery key** to AD or Entra ID. Lose that key and the data is gone.
- **Event Viewer** (`eventvwr.msc`): system, application, and security logs. Where root cause work starts.
- **Device Manager** (`devmgmt.msc`): driver and hardware detection issues
- **Services** (`services.msc`): starting and stopping services
- **Safe Mode**: booting with minimal drivers to isolate a problem
- **WSUS / Windows Update**: patch deployment and management

![An error entry in the Event Viewer log](https://ik.imagekit.io/stephanie/Blog-2/blog2-18.png?updatedAt=1789504978287)

Of all of these, Event Viewer is the one you'll open most. When a user tells you "it just doesn't work," the log has the exact timestamp and error code.

---

## 5. Accounts and Security

- **MFA (Multi-Factor Authentication)**: password plus a second factor. "I got a new phone and MFA stopped working" is a daily ticket.
- **SSO (Single Sign-On)**: one sign-in granting access to multiple systems
- **SAML / OAuth**: the protocols that implement SSO
- **Least Privilege**: granting only the access a role genuinely requires
- **Local Admin**: administrator rights valid on one machine only. As a rule, standard users don't get it.
- **UAC (User Account Control)**: the prompt that appears when an action needs elevation
- **Phishing**: the user reports it, support does the first review, and it gets **escalated** to the security team

---

## 6. Microsoft 365

![The Microsoft 365 admin interface](https://ik.imagekit.io/stephanie/Blog-2/blog2-19.png?updatedAt=1789504978450)

- **Exchange / Exchange Online**: the corporate mail system
- **OWA (Outlook Web App)**: Outlook in a browser. "Does it work in OWA?" is the standard question that separates a client-side problem from a server-side one.
- **Shared mailbox**: a mailbox several people access together (`support@company.com`)
- **Distribution list**: an email group
- **License assignment**: M365 licensing. Without it, the apps simply won't open.
- **OneDrive / SharePoint / Teams**: file sync and collaboration. Sync errors and permission problems make up most of the tickets.

---

## 7. Service Desk Process Terminology

This section matters more than any other in interviews.

- **Ticket**: the record of every request. Work that isn't documented didn't happen.
- **Incident**: something is **broken** (the printer isn't working)
- **Service Request**: a standard **ask** (a new monitor, a new account)
- **Problem**: the **underlying cause** behind recurring incidents
- **Change**: a planned modification, subject to an approval process
- **SLA (Service Level Agreement)**: the committed response and resolution times
- **Priority / Severity**: determined by impact and urgency. One user versus an entire department are not the same priority.
- **Triage**: sorting incoming tickets and assigning priority
- **Escalate**: passing a ticket up when it exceeds your access or knowledge. **Tier 1 → Tier 2 → Tier 3**. Job postings almost always include the phrase `escalate issues as needed`.
- **KB (Knowledge Base) article**: documented resolution steps. When the same ticket keeps coming back, you write one.
- **RCA (Root Cause Analysis)**: determining why it happened, not just how it was fixed
- **Onboarding / Offboarding**: setting up accounts and hardware for new hires, disabling accounts and recovering equipment for departures
- **Asset management**: tracking inventory and assignment history
- **ITIL**: the standard framework behind all of the above

![A ticket open in Jira showing priority, type, assignee, and work log](https://ik.imagekit.io/stephanie/Blog-2/blog2-20.png?updatedAt=1789504978260)

In a real ticket, all of those terms sit together on one screen. Priority, type, assignee, and the work log all live in the same place.