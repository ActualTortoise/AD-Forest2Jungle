# Auditing in Active Directory

## Introduction  
Active Directory has many powerful auditing capabilities built into it without the need for 3rd party products. The issue is most of this functionality is turned off by default and must be properly configured. Often when you need this telemetry and assume it was there, it's too late... There are many behaviours and interactions you can audit in Active Directory. Once again it is worth noting that every environment and use case is different so feel free to tweak and be creative in a way that will meet your requirements.

The amazing thing about auditing in Active Directory is that you are able to enable auditing for any object that has an access control list: group policy objects, user objects, file structures, certificate templates and more. If correctly leveraged you can use this telemetry to infer suspicious activity. For example; if you have auditing turned on for 5 unique and unused account objects in your domain and all flag a 4625 event (failed logon) over a short time period, this is almost a guarantee of password spraying in your environment. More of these inferences will be covered under the honeypots section.

I recommend familiarising yourself with the following documentation:  
- Advanced security auditing FAQ  
  https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/advanced-security-auditing-faq  
- Advanced security audit policy settings - Microsoft  
  https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/advanced-security-audit-policy-settings  

The whys of auditing AD elements are not covered in this technical guide. If you are curious as to the whys, whats and hows, you should read all sections under the Advanced Security Auditing FAQ by Microsoft.

The worst outcome here is a lot of unintended event log entries - as long as you are careful with your scoping you should be fine. Tweaking over time will be necessary, Rome wasn't built in a day!

---

## Phase 1 - Group Policy Setup  
Before the bulk of these auditing options even become available, you will need to configure and link some group policy objects.

I tend to create four different objects with the following scopes:  
- Domain Controllers  
- Servers  
- Clients  
- Certificate Services  

Depending on your AD environment you may want more objects or to enable even more auditing. I would recommend objects scoped on DC, Servers and Clients as a minimum.

### Domain Controller Auditing Policy Object:  
`Computer Configuration\Policies\Windows Settings\Security Settings\Advanced Audit Configuration`  

**Account Logon:**  
- Audit Credential Validation (Success, Failure)  
- Audit Kerberos Authentication Service (Success, Failure)  
- Audit Kerberos Service Ticket Operations (Success, Failure)  

**Account Management:**  
- Audit Computer Account Management (Success, Failure)  
- Audit Other Account Management Events (Success, Failure)  
- Audit Security Group Management (Success, Failure)  
- Audit User Account Management (Success, Failure)  

**Detailed Tracking:**  
- Audit DPAPI Activity (Success)  
- Audit Process Creation (Success, Failure)  

**DS Access:**  
- Audit Directory Service Access (Success, Failure)  
- Audit Directory Service Changes (Success, Failure)  

**Logon/Logoff:**  
- Audit Account Lockout (Success)  
- Audit Logoff (Success)  
- Audit Logon (Success/Failure)  
- Audit Other Logon/Logoff Events (Success, Failure)  
- Audit Special Logon (Success, Failure)  

**Policy Change:**  
- Audit Audit Policy Change (Success, Failure)  
- Audit Authentication Policy Change (Success)  

**Privilege Use:**  
- Audit Sensitive Privilege User (Success, Failure)  

**System:**  
- Audit IPsec Driver (Success, Failure)  
- Audit Security State Change (Success, Failure)  
- Audit Security System Extension (Success, Failure)  
- Audit System Integrity (Success, Failure)  

---

### Server Auditing Policy Object:  
`Computer Configuration\Policies\Windows Settings\Security Settings\Advanced Audit Configuration`  

**Account Logon:**  
- Audit Credential Validation (Success, Failure)  

**Account Management:**  
- Audit Computer Account Management (Success)  
- Audit Other Account Management Events (Success, Failure)  
- Audit Security Group Management (Success, Failure)  
- Audit User Account Management (Success, Failure)  

**Detailed Tracking:**  
- Audit DPAPI Activity (Success, Failure)  
- Audit Process Creation (Success, Failure)  

**Logon/Logoff:**  
- Audit Account Lockout (Success)  
- Audit Logoff (Success)  
- Audit Logon (Success, Failure)  
- Audit Special Logon (Success, Failure)  

**Object Access:**  
- Audit File System (Success, Failure)  

**Policy Change:**  
- Audit Audit Policy Change (Success, Failure)  
- Audit Authentication Policy Change (Success)  

**System:**  
- Audit IPsec Driver (Success, Failure)  
- Audit Security State Change (Success, Failure)  
- Audit Security System Extension (Success, Failure)  

---

### Client Auditing Policy Object:  
`Computer Configuration\Policies\Windows Settings\Security Settings\Advanced Audit Configuration`  

**Account Logon:**  
- Audit Credential Validation (Success, Failure)  

**Account Management:**  
- Audit Computer Account Management (Success)  
- Audit Other Account Management Events (Success, Failure)  
- Audit Security Group Management (Success, Failure)  
- Audit User Account Management (Success, Failure)  

**Detailed Tracking:**  
- Audit PNP Activity (Success)  
- Audit Process Creation (Success)  

**Logon/Logoff:**  
- Audit Account Lockout (Success)  
- Audit Group Membership (Success)  
- Audit Logoff (Success)  
- Audit Logon (Success, Failure)  
- Audit Other Logon/Logoff Events (Success, Failure)  
- Audit Special Logon (Success)  

**Object Access:**  
- Audit Detailed File Share (Failure)  
- Audit File Share (Success, Failure)  
- Audit Removable Storage (Success, Failure)  

**Policy Change:**  
- Audit Audit Policy Change (Success, Failure)  
- Audit Authentication Policy Change (Success)  
- Audit MPSSVC Rule-Level Policy Change (Success, Failure)  

**System:**  
- Audit IPsec Driver (Success, Failure)  
- Audit Security State Change (Success, Failure)  
- Audit Security System Extension (Success, Failure)  
- Audit System Integrity (Success, Failure)  

After these objects are prepared and ready to go, it is just a matter of linking them to their appropriate scopes.  

However, you are not done yet. In the majority of cases the above group policy settings only enables the ability to audit specified objects. You still need to configure the system access control list (SACL) for most of the objects you want to audit.

---

## Phase 2 - How to Configure System Access Control Lists  
The "access" subsections of the advanced audit configuration require that you set a system access control list on the objects you want to audit.

These subsections are:  
- DS Access (Active Directory objects)  
- Object Access (Files, File shares, DFS)  

This is so you are able to configure exactly what is audited to meet your needs. Everything is off by default. This is by design. Imagine a hypothetical situation where object access auditing was just switched on for your main file server used by several thousand users. Your event logs would blow before you could finish your coffee!

On that note, you do not have to worry when switching on Object Access and DS Access policies exactly because of this. Colleagues and others have expressed concern to me due to this before - nothing to worry about! Promise!

We will configure the system access control list on a dummy user object in Active Directory for this example. Configuring the system access control list is exactly the same regardless if we are working with the SACL on a file share, a user or another directory object. As a result, this example will be relevant for all objects you wish to audit. In this example we will be using AD Users and Computers, but use the tool you feel most confident with.

### Hypothetical Situation:  
We want to create an unused user object that will create an event log entry whenever it is viewed or read. No applications or users need access to this user, so if it is enumerated this could be a cause for concern.  
We create the user object and open the system access control list by right clicking on the object > Properties > Security > Advanced > Auditing.

We only want the event telemetry to trigger when the object is read, so we need to set the system access control list accordingly. Add a new permission under the auditing tab. Select **"Authenticated Users"** as the principal. Select **Clear All** under permissions and properties and choose the following:  
- List contents  
- Read all properties  
- Read permissions  

And save these changes. Wait for replication and then go in and read the object in question. Take a look for event 4662(S) on your domain controllers, you should see an entry along with an access mask which lets you know exactly which operation is being logged and who performed it. You can read the following documentation for more information on 4662:  
- [Event 4662 (S) - Microsoft](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4662)  

You can also change the access control list under auditing to audit for deletions, creations, writes and other specific scenarios to meet your needs.

---

## Phase 3 - Centralised Logging  
Before all of this telemetry can really start working for you, a form of centralised logging is best. Either a SIEM platform (Security Information and Event Management) or dedicated log platform. There are quite a few open source alternatives that are available without extra cost that you are able to run given you have the infrastructure available. Centralised logging has given our organisation so many advantages and has in many ways transformed a lot of the ways in which we work.

- Loki for general centralised logging  
  https://grafana.com/oss/loki/  
- Elastic as an open source SIEM solution  
  https://www.elastic.co/  

Initialisation of these platforms is not covered by this guide.

Alternatively - you can use event log PowerShell cmdlets to aggregate the telemetry you need on an adhoc basis, but this is not recommended.

Once you have set up your log ingestion, your AD auditing is up and running! You are now able to use this information to configure alerts and gain security insights.
