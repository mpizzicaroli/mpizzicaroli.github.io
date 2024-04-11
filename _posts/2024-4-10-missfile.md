---
layout: post
title: 'Misfile://'
---
Before I start, I want to give a shout to the Charles Schwab Threat Intelligence team and our leadership for giving me the opportunity and time to give this some legs. As the new Unstructured Hunt lead, this was a thrilling find. 

## Discovery

The team and I were discussing [the MonikerLink bug from CheckPoint](https://research.checkpoint.com/2024/the-risks-of-the-monikerlink-bug-in-microsoft-outlook-and-the-big-picture/) and whether or not you could downgrade the attack to WebDAV if SMB was blocked. We did some testing and determined nope, not possible.

![]({{site.baseurl}}/images/monikerfail.png)
<img src="/images/monikerfail_outcome.png" width="1000" style="float: center;"/>
![]({{site.baseurl}}/images/monikerfail_outcome.png)
![]({{site.baseurl}}/images/monikernohash.png)

We knew it had been patched otherwise, and we weren’t exposed, and to make it worse, our good Attack Surface Management [SME dropped this lil guy on us](https://www.microsoft.com/en-us/security/blog/2023/03/24/guidance-for-investigating-attacks-using-cve-2023-23397/):

> Interaction based on the WebDAV protocol is not at risk of leaking credentials to external IP addresses via this exploit technique. While the threat actor infrastructure might request Net-NTLMv2 authentication, Windows will honor the defined internet security zones and will not send Net-NTLMv2 hashes. In other words, an external threat actor can only exploit this vulnerability via the SMB protocol.

So I said fineeeee, lets check some Microsoft Outlook versions just in case. I noted mine, threw some queries into our SIEM, and noticed something odd. Version numbers I didn’t recognize. I sanity checked myself by pestering a colleague and he hit me with: “1.2024.214.400”, not version ‘WXYZ’ like I expected.

<img src="/images/huhcat.gif" width="100" style="float: center;"/>

This led me to a wonderful new discovery – there is a “New Outlook”…and I’ve simply been ignoring the option in the top right corner this entire time.

![]({{site.baseurl}}/images/travolta.png)

Alright fair play Microsoft. I attempted to find the security release notes for this “New Outlook”
![]({{site.baseurl}}/images/feb.png)

And realized I can't find them. I still to this day can’t find those so Microsoft if you have them please tell me!

So I said fine, let’s try MonikerLink in New Outlook and see if we can get an NTLM hash leak. I fully expected it to fail…and it kind of did.

## Testing

Instead of getting the error I expected like thick Outlook, 
![]({{site.baseurl}}/images/error_okay.png)
New Outlook asked if I wanted to open the file…but there was no hash leak…so no preview pane unfortunately.
![]({{site.baseurl}}/images/continue.png)