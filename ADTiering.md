# Tiering in Active Directory

## Introduction

**Resources:**  
https://itm8.com/articles/fundamentals-ad-tiering

Tiering in Active Directory is one of the most effective security measures to increase your overall posture.

In short, it consists of placing all of your Active Directory objects into a designated tier. The aim is to completely segregate the different tiers from each other, making privilege escalation much more difficult.

The standard is to divide an AD environment into 3 different tiers:

### Tier 0 - Control of Environment  
As a rule highly privileged objects in your domain. If the answer to "can I take over my domain with this?" is yes, then it needs to be designated as tier 0.

### Tier 1 - Apps, Servers, Data and Services  
The majority of your applications, shares, datastores, databases, and services go here.  
If the answer to the question above is no, and the answer to the question "Does this object play a part in providing a service or running an application?" is yes, then it should be designated as tier 1.

### Tier 2 - Client machines and standard users  
If the answer to both of the questions above is no, it goes here.

---

There are many different ways to implement tiering. It can even be appropriate to have more than 3 tiers depending on your environment. Each tiering technical guide tends to have its own way of doing things.

I will explain my reasoning behind certain choices and you can determine whether or not it is appropriate for your environment.

---

## Phase 1 - Setting the foundations

First off, you will want to decide your strictness. Some tier model implementations allow for **downwards mobility** — in other words, user accounts designated as tier 0 have the ability to authenticate with tier 1 and tier 2 systems.

While this is practical, in my implementations I have **not allowed for downwards mobility**. This is due to the fact that anyone sitting on a machine with local administrator rights can dump your authentication information and use it.

Therefore, if an attacker compromises a machine in tier 2 or tier 1 and lays a trap, trying to provoke a log on from a tier 0 account... suddenly they own the domain.

Imagine an attacker has compromised a print server and turned off a vital print service. You log in with your tier 0 administrator account to resolve the issue and that is it. It is much better to completely segregate the tiers from each other.

---

### General recommendations for privileged access management for each tier:

- **Tier 0** - Domain Administrator account set - logon restricted exclusively to tier 0.
- **Tier 1** - Administrator account set - logon restricted exclusively to tier 1 (local admin permissions for tier 1, no tier 0 permissions!).
- **Tier 2** - Windows Local Administrator Password Solution (LAPS)  
  [LAPS Overview](https://learn.microsoft.com/en-us/windows-server/identity/laps/laps-overview)

---

Start by making an organisational unit (OU) hierarchy in your domain where this is necessary. How you do this may differ depending on how your environment is set up and where you store your objects.

**Some examples:**

- Under the **Servers** organisational unit you can create Tier 0, Tier 1 and Tier 2 containers to sort your servers.
- If you store your client machine objects in a **Computer** organisational unit, it may not be necessary to create a tier hierarchy there. As standard clients are inherently tier 2 (except privileged access workstations).
- The **Domain Controllers** organisational unit is another example that does not need tiering. Everything here is inherently tier 0.
- For users you will also want to create a Tier 0, Tier 1 and Tier 2 hierarchy similar to servers. If you already store your privileged accounts separately from the rest, you can skip this.

The most important part is that you have infrastructure in place in your Active Directory environment that allows for the isolation of tiers. As a rule, a container should only contain computer and user objects from one tier.

If this is awkward or difficult to do for your environment, you can also use security groups for GPO filtering instead. There are many different methods to achieve tier isolation.

---

Once this infrastructure is in place in your environment, you can start sorting your objects. Find some concrete examples below:

- **Tier 0:** Domain Administrators, Domain Controllers, Certificate Authorities, System Center, DNS, Management Servers, Exchange, Privileged Access Workstations, and more.
- **Tier 1:** Databases, File Servers, Application Servers, Internal Web Services, Tier 1 Administrators, Tier 1 Users and Vendors, Services, Automation, and more.
- **Tier 2:** Client machines, standard user accounts, non-sensitive testing environments.

Again, what object belongs to which tier depends on your own environment and needs. It is worth taking a discussion internally to decide what should be placed where during this phase.

---

Something also worth considering is whether to further isolate machines running services frequently accessed by external consultants or vendors to mitigate the effects of a supply chain attack.

Administrative access to these machines would be managed via Windows Local Administrator Password Solution and would be tier 1.5 in practice.

---

When this is done, you will want to create the following security groups and populate them:

- `TIER0_USERS`
- `TIER0_COMPUTERS`
- `TIER1_USERS`
- `TIER1_COMPUTERS`
- `TIER2_USERS`
- `TIER2_COMPUTERS`

---

## Phase 2 - Designing and implementing your Group Policy Objects (GPOs)

This phase will vary depending on your environment and needs. You may need to make your own group policy exceptions out of necessity. Trial and error is key during this phase, and make sure *not* to deploy this to production until it has been thoroughly tested!

### The three core GPOs are as follows:

#### TIER_0_ISOLATE

**Defined Settings:**  
`Computer Configuration\Policies\Windows Settings\Security Settings\Local Policies\User Rights Assignment`

**Policies:**  
- Deny Access to this computer from the network: `BUILTIN\Guests`  
- Deny log on as a batch job: `TIER1_USERS`, `TIER1_COMPUTERS`, `TIER2_USERS`, `TIER2_COMPUTERS`  
- Deny log on as a service: `TIER1_USERS`, `TIER1_COMPUTERS`, `TIER2_USERS`, `TIER2_COMPUTERS`  
- Deny log on locally: `TIER1_USERS`, `TIER1_COMPUTERS`, `TIER2_USERS`, `TIER2_COMPUTERS`  
- Deny log on through Terminal Services: `TIER1_USERS`, `TIER1_COMPUTERS`, `TIER2_USERS`, `TIER2_COMPUTERS`

---

#### TIER_1_ISOLATE

**Defined Settings:**  
`Computer Configuration\Policies\Windows Settings\Security Settings\Local Policies\User Rights Assignment`

**Policies:**  
- Deny Access to this computer from the network: `TIER0_USERS`, `TIER0_COMPUTERS`, `BUILTIN\Guests`  
- Deny log on as a batch job: `TIER0_USERS`, `TIER0_COMPUTERS`, `TIER2_USERS`, `TIER2_COMPUTERS`  
- Deny log on as a service: `TIER0_USERS`, `TIER0_COMPUTERS`, `TIER2_USERS`, `TIER2_COMPUTERS`  
- Deny log on locally: `TIER0_USERS`, `TIER0_COMPUTERS`, `TIER2_USERS`, `TIER2_COMPUTERS`  
- Deny log on through Terminal Services: `TIER0_USERS`, `TIER0_COMPUTERS`, `TIER2_USERS`, `TIER2_COMPUTERS`

---

#### TIER_2_ISOLATE

**Defined Settings:**  
`Computer Configuration\Policies\Windows Settings\Security Settings\Local Policies\User Rights Assignment`

**Policies:**  
- Deny Access to this computer from the network: `TIER1_USERS`, `TIER0_USERS`, `TIER0_COMPUTERS`, `BUILTIN\Guests`  
- Deny log on as a batch job: `TIER0_USERS`, `TIER0_COMPUTERS`, `TIER1_USERS`  
- Deny log on as a service: `TIER0_USERS`, `TIER0_COMPUTERS`, `TIER2_USERS`  
- Deny log on locally: `TIER0_USERS`, `TIER0_COMPUTERS`, `TIER2_USERS`  
- Deny log on through Terminal Services: `TIER0_USERS`, `TIER0_COMPUTERS`, `TIER2_USERS`

---

Once the GPOs are in place, link them to your designated computer containers:

- `TIER_0_ISOLATE` linked to, for example, domain controllers and `Servers\Tier 0`
- `TIER_1_ISOLATE` linked to `Servers\Tier 1`
- `TIER_2_ISOLATE` linked to client machines

Depending on your environment, you will have to make changes until you can confirm everything works as intended. Once this has been successfully implemented, you now have tiering in your environment.

---

## Phase 3 - Tightening the bolts

Now that we have implemented tiering, we need to see if our privileges are correctly configured for each tier. You should follow up with a revision of this every three months or so.

For this task, I use a tool called **Forest Druid**, the same one mentioned in the in-house penetration testing segment. This tool is free and developed by Semperis:  
https://www.semperis.com/forest-druid/request-form/

In Forest Druid, every object can be given a designated tier. Once every object has been properly categorised in the tool, you can then see if there is any privilege overlapping between your tiers.

For example, have any tier 2 user accounts been given privileges that should be restricted to tier 1? Or have any tier 1 accounts been given tier 0 privileges? Forest Druid will help you map this out effectively; a task that would be a nightmare otherwise.

Once you have carried out your spring cleaning you can then enjoy a properly tiered Active Directory environment!
