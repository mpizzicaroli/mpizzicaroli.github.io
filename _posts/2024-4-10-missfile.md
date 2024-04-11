---
layout: post
title: 'Missfile://CVE-2024-20670'
---
Before I start, I want to give a shout to the Charles Schwab Threat Intelligence team and our leadership for giving me the opportunity, time, and opinions to give this some legs. As the new Unstructured Hunt lead, this was a thrilling find. 

## Discovery

The team and I were discussing [the MonikerLink bug from CheckPoint](https://research.checkpoint.com/2024/the-risks-of-the-monikerlink-bug-in-microsoft-outlook-and-the-big-picture/) and whether or not you could downgrade the attack to WebDAV if SMB was blocked. We did some testing and determined nope, not possible.

![]({{site.baseurl}}/images/monikerfail.png)
![]({{site.baseurl}}/images/monikerfail_outcome.png)
![]({{site.baseurl}}/images/monikernohash.png)

We knew it had been patched otherwise, and we weren’t exposed, and to make it worse, our good ol' Attack Surface Management SME [dropped this lil guy on us](https://www.microsoft.com/en-us/security/blog/2023/03/24/guidance-for-investigating-attacks-using-cve-2023-23397/):

> Interaction based on the WebDAV protocol is not at risk of leaking credentials to external IP addresses via this exploit technique. While the threat actor infrastructure might request Net-NTLMv2 authentication, Windows will honor the defined internet security zones and will not send Net-NTLMv2 hashes. In other words, an external threat actor can only exploit this vulnerability via the SMB protocol.

So I said _fineeeee_, lets check some Microsoft Outlook versions just in case something got missed. I noted mine, threw some queries into our SIEM, and noticed something odd. Version numbers I didn’t recognize. I sanity checked myself by pestering a colleague and he hit me with: “1.2024.214.400”, not version ‘WXYZ’ like I expected.

![]({{site.baseurl}}/images/huhcat.gif)
  
This led me to a wonderful new discovery – there is a “New Outlook”…and I’ve simply been ignoring the option in the top right corner this entire time.

![]({{site.baseurl}}/images/travolta.png)

Alright fair play Microsoft. I attempted to find the security release notes for this “New Outlook”

![]({{site.baseurl}}/images/feb.png)

And realized I can't find them. I still to this day can’t find those so Microsoft if you have them please tell me!

So I said _fineeeee_ again, let’s try MonikerLink in New Outlook and see if we can get an NTLM hash leak. I fully expected it to fail…and _it kind of did?_

## Testing

Instead of getting the error I expected like thick Outlook,

![]({{site.baseurl}}/images/error_okay.png)

New Outlook asked if I wanted to open the file…but there was no hash leak…so no preview pane vulnerability unfortunately.

![]({{site.baseurl}}/images/continue.png)

Playing around for a bit and running through Wireshark, I noticed something interesting: the NetBIOS domain and computer name definitely weren’t my labs.

![]({{site.baseurl}}/images/notmypc.png)

I began sleuthing the internet (read: googling for 5 minutes) and found that local file paths didn’t work in New Outlook, and probably as a work around, [file:// was hacked into New Outlook](https://answers.microsoft.com/en-us/outlook_com/forum/all/new-outlook-365-hyperlinking-a-local-file/f46f71ba-a1cb-4c3d-ab84-be9f88984c64)

After more research and testing, what I had found was that when you sent the file:// path in a link in an email, New Outlook was automatically appending it to the following url, likely due to SafeLinks.

![]({{site.baseurl}}/images/outlooklink.png)

I captured a PCAP but didn’t do any break and inspect. I did notate traffic to Microsoft, and then a response from Microsoft. My theory is that since New Outlook is essentially OWA (Outlook On The Web), SafeLinks checks the domain, sees it’s a file:// call, and returns it back to the user.

![]({{site.baseurl}}/images/email_one.png)

There's no need to play with OLEs, Monikers, anything... it just accepts file:// outright.

![]({{site.baseurl}}/images/unsafe.png)

So, if you could get a user to enter credentials, you could get a NTLM hash leak.

<p float="center">
<img src="/images/dummycreds.png" width="500" style="float: center;"/>
<img src="/images/realhumanbean.png" width="500" style="float: center;"/>
</p>

Quite a few clicks, but it was doable. At this point, I opened a case with Microsoft for responsible disclosure that you could leak NTLM hashes with file:// links in New Outlook.

While I waited for a case, I continued to play with this vulnerability and I realized, that since we’re truly using file://, you can downgrade to WebDAV, which also means you could probably get someone to download a file!

So back to 80. 

![]({{site.baseurl}}/images/80works.png)

And I got stuck. I spent a day trying to get New Outlook to download the file, and all I could get was propfinds and options. I'm sure someone with more experience with the WebDAV protocol could do something clever.

![]({{site.baseurl}}/images/wireshark.png)

But while I was reading about file:// and WebDAVs interactions, I eventually realized I was staring at a tree in a forest. After all, file protocol interacts with the local system right? Let’s see if we can open Powershell.

![]({{site.baseurl}}/images/powershellemail.png)
![]({{site.baseurl}}/images/runpowershell.png)
![]({{site.baseurl}}/images/gotem.png)

I updated the case with Microsoft with these further findings that you could remotely launch any executable on a host and stopped my investigation here. I know it’s possible to provide arguments in Windows Explorer, but I was unable to get any further and did have a day job to do. I’m sure there’s some clever UNC pathing that could be done, but for me, an NTLM hash leak and a remote file execute in one vulnerability was enough.

## Outcome

![]({{site.baseurl}}/images/timeline.JPG)

[CVE-2024-20670](https://msrc.microsoft.com/update-guide/en-US/vulnerability/CVE-2024-20670)

CVSS:3.1 Score - 8.1 / 7.1 

I can't say I set out to find a CVE, but I'm **very happy** to hang my first hat on an 8.1.
