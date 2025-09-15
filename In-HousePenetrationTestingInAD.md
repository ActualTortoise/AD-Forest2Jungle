# In-house Penetration Testing (AD)

## Introduction  
In-house penetration testing is a great way to improve your organisation’s security posture over time.  
If you do it regularly and in a consistent manner while also documenting your results, not only do you have something to show management, but you also end up with a great overview of your environment. You also gain insights into how attackers think.

Many of the tools used in penetration testing exercises are also utilised by attackers.  
By carrying out regular in-house penetration tests, you can close vulnerabilities, misconfigurations and privileges you find before they are abused.

If you are part of an organisation that regularly hires consultants to perform penetration tests in your environment, they can spend their time looking for vulnerabilities that are harder to identify.  
From the red teamers I have talked to, the majority of their finds are low hanging fruit which could be identified and closed through tool-based in-house testing. Make them earn that money!

> It is worth noting that this guide will only cover Active Directory relevant penetration testing and tools. Network scanning methodologies and vulnerable service tests will not be included here, but are worth considering for your overall plan.

**Important:** When downloading and deploying these tools make sure to check the hashes are correct and match documentation. There are bad actors who often attempt to deploy malicious versions of the tools.

---

## Designing your In-House Penetration Test  
When putting your test together, there are a few things you want to think about:  
- Will including a tool or activity give you actionable information?  
- Is it possible to put together a results report from a tool or activity that can demonstrate changes over time?  
- What scheduling is most appropriate for your organisation? 3 months? 6 months? annually?  
- When receiving actionable information from tests, is there a process in place to make sure this is followed up?  
- Do you have a process for documenting made changes and managing risk?  
- How much time and money are you willing to spend?  

You should then be ready to put together a framework of tools and activities that will form the basis of your in-house penetration test.  

> If you have implemented AdminSDHolder obfuscation in your environment, you will need to run these with an account that is a member of the reader group. If not, some of these tools will make a lot of noise and be mostly useless.

---

## Recommended Tools

### PingCastle - Netwrix  
[https://www.pingcastle.com/download/](https://www.pingcastle.com/download/)  
Used to discover insecure configurations and vulnerabilities in your domain. Provides easy to understand domain score metrics so that it is easy to monitor improvements over time.  

One of the best elements of PingCastle is that for every issue that is brought up, a concrete remediation guide is included. Each issue also has its own severity grade making it easier to prioritise.  
A very good start!

---

### PurpleKnight - Semperis  
[https://www.semperis.com/purple-knight/request-form/](https://www.semperis.com/purple-knight/request-form/)  
Used to discover insecure configurations and vulnerabilities in a similar fashion to PingCastle, with a greater focus on certificate, group policy and encryption type configurations.  

You will get a grade determining your environment health making it easy to track progress over time. In addition this tool also includes remediation advice alongside all of its discoveries. You should also have the ability to run the tool against your EntraID environment if it is relevant for your organisation to include in the scope.

---

### ForestDruid - Semperis  
[https://www.semperis.com/forest-druid/request-form/](https://www.semperis.com/forest-druid/request-form/)  

You can use either ForestDruid or BloodHound depending on what your organisation has the most experience with and is most comfortable using. ForestDruid has an easy to use GUI whereas BloodHound is more easily leveraged from dedicated pentesting operating systems such as Kali.  

The aim with ForestDruid is to uncover any and all paths to Tier 0/Domain admin. If your domain is as old as some of the ones I have worked with - you'll find something.  

- An account with way too many rights that was granted them because it just worked during a deployment, and was promptly forgotten.  
- Misinterpretation of a security group or privilege that actually grants far more access than intended.  
- Old objects from when standards were vastly different from current ones.  

These tools will help you uncover all of the most egregious cases and clean up even the most tangled of environments.  

Make sure to document any and all changes when working with these tools in case you remove any privileges that were actually required for services in your environment. You can easily leverage centralised logging to determine if an object is still active.

---

### Snaffler - l0ss/Sh3r4  
[https://github.com/SnaffCon/Snaffler](https://github.com/SnaffCon/Snaffler)  

Snaffler is a great tool for enumerating all shares in an environment and which ones that a chosen user has access to. There could be several shares on the network that normal users have access to, even if they are not mapped or hidden under normal circumstances.  

Snaffler will find these shares for you. In addition you can define things you want it to look for. It is able to search the contents of `.txt` files, `.jsons`, `.xmls` etc., to see if there is any sensitive data lying around on freely accessible shares in the form of passwords, secrets and keys.

You can configure these rules yourself to your heart's content to design a Snaffler test that best suits the needs of your environment. Recommend reading the Github readme to get the most out of it.

---

### hashcat  
- [https://hashcat.net/hashcat/](https://hashcat.net/hashcat/)  
- [https://www.netwrix.com/ntds_dit_security_active_directory.html](https://www.netwrix.com/ntds_dit_security_active_directory.html)  

Hashcat is used for cracking password hashes. It is best to run this on a secured machine with enough compute to be able to crack in a reasonable timeframe. The reasoning behind this is that if you can crack your users’ passwords, an intruder sure can too.

You can then inform the users in question that they will need to change their password. A good starting point is running one rule in combination with the latest lightweight rockyou wordlist. You will also need to extract the NTLM hash database. Netwrix has a good write up on this—follow the link above.
