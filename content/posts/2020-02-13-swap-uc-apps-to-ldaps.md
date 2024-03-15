---
title: UC Swapping to LDAPS
author: Will
type: posts
date: 2020-02-14T02:12:00+00:00
url: /2020/02/swap-uc-apps-to-ldaps/
categories:
  - Cisco Unified Communications
  - Linux
  - Operating Systems
  - voice
summary: |
  After reading this, look at the bug I discovered when enabling secure LDAP on UC applications [here][1].

  As many of you are aware, Microsoft began the process to deprecate LDAP access into Active Directory back in March. You can read cisco's advisory here: [https://www.cisco.com/c/en/us/td/docs/voice\_ip\_comm/cucm/trouble/12\_5\_1/fieldNotice/cucm\_b\_fn-secure-ldap-mandatory-ad.html][2]

  Basically, this means we need to do a fairly simple swap from LDAP to LDAPS. I just completed one for a client's CUC and CUCM and it took about 30 minutes start to finish.
---
Due to the data loss, I had to recreate this, the wording may be different from what you recall but the steps remain the same.

After reading this, look at the bug I discovered when enabling secure LDAP on UC applications [here][1].

As many of you are aware, Microsoft began the process to deprecate LDAP access into Active Directory back in March. You can read cisco's advisory here: [https://www.cisco.com/c/en/us/td/docs/voice\_ip\_comm/cucm/trouble/12\_5\_1/fieldNotice/cucm\_b\_fn-secure-ldap-mandatory-ad.html][2]

Basically, this means we need to do a fairly simple swap from LDAP to LDAPS. I just completed one for a client's CUC and CUCM and it took about 30 minutes start to finish.

The first thing I did was download a linux ISO and deploy it into the customer's esxi environment. I used debian 10.4 because it's my goto lately, the netinst iso is around 330MB, of course you could use something like alpine which is even smaller. Once I had linux installed, you just need to grab the certs from the AD CA. To do this, use the following command 

`openssl s_client -showcerts -connect ip.of.AD.node:636`

![](/images/swap-uc-apps-to-ldaps/getcert-707x1024.png)

Once you've grabbed the certs, you need to copy the section starting with -----BEGIN and ending with -----END, as shown below. Don't leave out the dashes.

![](/images/swap-uc-apps-to-ldaps/2.png)

Save this file as something like ad.crt and then load it into your call manager or unity connection as a tomcast-trust certificate.

![](/images/swap-uc-apps-to-ldaps/ldaps1.png) 

Once you've loaded the certificate, you will be notified that you need to restart Cisco Tomcat, simply use the below command via SSH (requires your Platform Administration credentials)

`utils service restart Cisco Tomcat`

![](/images/swap-uc-apps-to-ldaps/ldaps2.png) 

Now you can go back to CM Administration and configure your Directory and Authentication sections. Simply change the ldap port to 636 and click use TLS. Click save and check for any error, if you see no errors you are good to go. You are looking for "Update Successful."

![](/images/swap-uc-apps-to-ldaps/ldaps3.png) 
![](/images/swap-uc-apps-to-ldaps/ldaps4.png)

**NOTE:** Sometimes you may run into a known bug, [CSCvc30013][3]. It states that it only affects Call manager on v12.0, but it applies to 10.5 up through 12.0 and also applies to Unity connection. I did not experience the bug in 12.5. Realistically, I'm not sure this is even a bug, To leverage this, we need to trust the certs which have an FQDN on them, yet we're connecting with an IP address, well, obviously the IP isn't trusted. I would say this "fix" is a bandaid to a mistake, not a fix for a problem. Also keep in mind, the bug only affects LDAP Authentication, it will sync just fine, you just can't login as an LDAP user. Either way, if you want to securely connect via IP address, as I've shown in my examples here, simply login to the affected devices and put 

`utils ldap config IPAddr`

Once you do this, everything will work fine, just ensure you apply this change to all of your nodes (including IM&P/CUPS nodes), or simply use the FQDN of your AD nodes. 

Below is an animated gif that shows the basic process from start to finish. I hope this helps!

![](/images/swap-uc-apps-to-ldaps/ldaps-full.gif)

 [1]: https://blog.longoconsulting.us/2021/04/secure-ldap-bug/
 [2]: https://www.cisco.com/c/en/us/td/docs/voice_ip_comm/cucm/trouble/12_5_1/fieldNotice/cucm_b_fn-secure-ldap-mandatory-ad.html
 [3]: https://bst.cloudapps.cisco.com/bugsearch/bug/CSCvc30013/?rfs=qvred