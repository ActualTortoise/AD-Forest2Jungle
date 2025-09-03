# Honeypots in Active Directory

## Introduction  
Before you will be able to utilise Honeypots in your environment, you will need to have enabled auditing.  
Please follow the auditing write up if you have not done this already.  

Ideally, you will also have access to a SIEM or another type of centralised logging. Monitoring honeypots is possible without this, via scripting, scheduled tasks and perhaps operations manager. It is a lot more work however and not as effective. Only recommended if you really have no other option.

What makes an effective honeypot in an AD environment may differ depending on your organisation. It is worth thinking from the attackers perspective, what resources do they want? Not only in regards to privilege escalation but also data exfiltration.

If you have a method for securing these resources while simultaneously presenting low hanging fruit in their place - you are well on your way to an effective honeypot!

There are many things to consider when optimising honeypots but you should aim to maximise confidence. In other words if someone triggers one of your honeypots, you want to be as certain as possible that it is due to suspicious activity. Here are some tips to consider to maximise confidence:  
- Is the honeypot something the organisation will legitimately access in the course of normal operations?  
- Is the honeypot located in an area that would require specific intent to access?  
- Is it possible to combine one or more honeypots to increase confidence? For example; automated tools collect different object types in a short timeframe.  
- Is the honeypot an object the attacker would be interested in to further their goals? For example; a file called `passwordlistIT.xlsx` or a privileged object.  

The only limit is your creativity, as every object in an Active Directory environment with an ACL can be potentially turned into a honeypot.

### Potential honeypot objects:  
- Users  
- Groups  
- Computers  
- Managed Service Accounts  
- Certificates  
- Certificate Templates  
- Object Containers  
- File Shares  
- Files  
- Group Policy Objects  

From what I have seen from red teamers, they often aim for privilege escalation through misconfigured certificates, group policies or overprovisioning of privileges. Therefore having a few honeypots lying in wait while they sniff around can be a very quick win.

---

## Phase 1 - Initialising the Honeypots  
For this write up we will be using the following example: we want a method to detect when someone is running PingCastle or a similar enumeration tool from anywhere in the domain.

To achieve this, we will create three honeypots: a user object, a group object and a computer object. We will then configure an alert that will trigger when all three pots are triggered in a given timeframe.

To prepare the honeypots, create one of each and place them in a suitable location. After this is done, follow Phase 2 under the auditing section and configure each object to trigger a 4662 alert when it is listed or read.

---

## Phase 2 - Testing and Documentation  
Now the honeypots are ready - we need to test them to see if they behave the way we want.  

- Read/list the honeypots either through the CLI (`Get-ADUser`/`Get-ADGroup` etc) or through ADUC/ADAC. Document the time.  
- Search for 4662 events triggered by your account either through central logging or in event viewer on your domain controller(s). In the event message, you want to locate the **Object Name** value and document this.  

The object name is obscured (but static). This will be in the form of a GUID with the following format: `%{XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX}`. You will want to document each of your honeypots GUIDs as you will use these to collate your alerts.

Once you are satisfied that all your honeypots are behaving as intended, you can proceed to Phase 3.

---

## Phase 3 - Alert Setup, Testing and Streamlining  
Now it is time to set up the alert - this is best done within your SIEM or centralised logging system. This step will differ greatly depending on your system of choice but I will provide you with the algorithm logic.

Within the span of 180 seconds:  
If the following 3 events occur:  
1. Where event action `"Directory Service Access"` and event object name `"HoneypotGUID1"`  
2. Where event action `"Directory Service Access"` and event object name `"HoneypotGUID2"`  
3. Where event action `"Directory Service Access"` and event object name `"HoneypotGUID3"`  

**and**  
Event subject username is identical  
**Trigger alert**

This makes it so that if a single user accesses all three honeypots in a given time frame it will trigger an alert. The confidence level on this one should be high.

After the alerting rule is in place, you should audit it for 30 days to see if there are any legitimate use triggers.  
You may have an automated account management system that enumerates your Active Directory regularly for example. In that case you will want to create an exception for it, or any similar services.

You will also want to test the alert to see if it is fit for purpose. Run PingCastle or another enumeration tool of your choice. Check to see if the alert triggers as intended. If not you may have to change your alert logic.

Once you are satisfied, you can place the alerting rule into production.

---

## Phase 4 - Continuous Improvement and Frequent Testing  
It is likely that even after a honeypot alerting rule is put into production, further changes may need to be made in regards to adding exceptions or perhaps tweaking span times.

As your environment changes, alerting rules may stop working or become defunct. Try to test these rules and trigger them during your in-house penetration tests to see they are still functioning as intended.

The same logic should hold true for honeypots of all object types in Active Directory!  

Best of luck!
