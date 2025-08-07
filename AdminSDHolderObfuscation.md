# AdminSDHolder Obfuscation

## Introduction

Before starting implementation of AdminSDHolder obfuscation in your environment I strongly recommend reading and familiarising yourself with the following:

- [Improving AD Security Posture: AdminSDHolder](https://www.semperis.com/wp-content/uploads/resources-pdfs/resources-improving-ad-security-posture-adminsdholder.pdf)
- [Protected Accounts and Groups in Active Directory (Protected Groups and SDProp)](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/appendix-c--protected-accounts-and-groups-in-active-directory)

Every AD environment is different, what has worked well for me may not meet your requirements. You may make changes to your AD Schema to better reflect your needs, given there are more groups you would like to protect. You can also manually protect an object by setting its `adminCount` attribute to `1`. In the example used here we have a single enterprise sized domain in a single forest.

Implementing this incorrectly could break your environment in so far that critical resources may no longer be able to read or find privileged objects. Always bear this in mind and proceed with caution. I bear no responsibility for any damage that occurs in an attempt to implement this security measure. Always err on the side of caution if you are unsure as to whether or not something needs read access to an obfuscated object.

This measure must be applied across the entire AD forest. Make sure the forest level is able to read critical resources from the domain levels and vice versa. Cross domain as needed.

---

## Phase 1 - Taking Stock and Cleaning Up

First off it is important we get an overview over which objects will be obfuscated when this change is implemented.

Run the following PowerShell cmdlet to get a quick overview of all objects in your environment that have their `adminCount` attribute set to `1`. These are the objects that have their access control lists managed by the SDProp process. The Active Directory module is a prerequisite.

**For users:**
```powershell
Get-ADUser -filter 'adminCount -eq 1' | Out-GridView
```

**For groups** (this should be the group list under appendix c, but you never know if your environment is old):
```powershell
Get-ADGroup -filter 'adminCount -eq 1' | Out-GridView
```

It is worth noting that when an account is removed from a privileged group, the `adminCount` attribute is not set back to `0`. This must be done manually. You can either do this in the interface, or via PowerShell:
```powershell
Set-ADUser -Identity samaccountnamehere -Remove @{adminCount="1"}
```

If the object that has its `adminCount` attribute removed is still in a privileged group listed under appendix c, it will be restamped the next time SDProp runs. It must also be removed from the associated privileged groups.

Continue cleaning up until all that remains are the objects you need to be obfuscated. Document these remaining objects using the previous PowerShell cmdlets and keep them in mind for future phases.

---

## Phase 2 - Initialising the Reader Security Group

Create a domain local security group and give it a name in compliance with your organisations naming conventions. Document the group along with its purpose if you are able.

We will use the name **"ASDH-Readers"** in this example. Create this group on the forest level, as well as for each domain in your forest.

Go into the security access control list for the AdminSDHolder object. Add the security group ASDH-Readers to the security access control list. Give it the same set of rights that the **"Everyone"** object is currently assigned:  
- List contents  
- Read all properties  
- Read Permissions  

Do this for the forest and every domain.

I did this via AD Users and Computers, but feel free to use PowerShell, LDP.exe, ADSI edit or your a tool of choice.

---

## Phase 3 - Populating the Reader Security Group

Here are a few objects that as a rule need to be members:

- Domain Admins
- Domain Controllers
- Enterprise Admins (Forest)
- System Center related groups and machine objects (if applicable)
- Exchange related groups and machine objects (if applicable)

If you suspect that certain objects may need read access, you should add those too.

Next, we want to initialise auditing for all objects that currently have their `adminCount` set to 1. This is not something enabled by default and must be configured via group policy and the access control list. Please follow the [audit](https://github.com/ActualTortoise/AD-Forest2Jungle/blob/main/AuditingInActiveDirectory.md) guide in the repository before continuing.

This should be possible to do without centralised logging. Although, many of my implementations have been with some form of centralised logging. In the event that your organisation has no form of centralised logging, you should strongly consider implementing this. Otherwise for this next step you will have to filter through your domain controllers event logs using PowerShell.

You will want to edit the **auditing acl** for all objects that still have their `adminCount` set to 1. Here we want to give **Authenticated Users** the List contents, Read all properties and Read permissions under the **Audit ACL**. Here you will want to catalogue a list of service accounts, machine accounts and other objects that require legitimate read access. This way you can almost guarantee continued service when implementing this into production.

We are interested in documenting the following events:

- **4624(s)** An account was successfully logged on  
  https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4624
- **4662(s)** An operation was performed on an object  
  https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4662

4624 will trigger whenever a privileged object authenticates. You will want to take note of all machine objects that appear here and add them to the Reader Security Group.

Example: 4624 occurs when a privileged account logs into the System Center Configuration Manager host - **SCCM01$**. Add **SCCM01$** to the reader security group to maintain current levels of service.

Do this for each machine that needs it. Alternatively - if you have implemented AD tiering in your environment, add all Tier 0 machine accounts to the reader security group.

4662 will trigger whenever an object reads one of your privileged objects if you have set up your auditing correctly. Keep in mind 4662 auditing may be used other places in your environment. Scope the 4662 search only to objects with their `adminCount=1`.

All non-standard user accounts that trigger 4662 and have a seemingly legitimate need to read privileged objects should be added to the Reader Security Group.

I would recommend running this audit for a minimum of 30 days before putting anything into production.

After the auditing phase is over and your reader security groups have been fully populated, you can continue to phase 4.

---

## Phase 4 - Moving into Production

If you have followed all the steps outlined above, you should be ready to move into production without this negatively affecting any of your running services.

Simply go into the access control list of AdminSDHolder and remove read rights from the **"Everyone"** object.

Wait for replication and try running a PowerShell cmdlet to enumerate for example Domain Admins with a non-privileged account. You should be told it doesn't exist!

**Congratulations!** You have just made your AD environment considerably more difficult to abuse.
