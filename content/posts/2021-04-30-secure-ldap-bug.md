---
title: Cisco UC – Secure LDAP bug
author: Will
type: posts
date: 2021-04-30T13:52:00+00:00
slug: secure-ldap-bug
aliases: /2021/04/secure-ldap-bug/
categories:
  - Cisco Unified Communications
  - voice
tags:
  - Cisco
  - LDAP
  - Unified Communications
summary: While working with a client this week, I encountered an undocumented bug with secure LDAP authentication. My client was doing an upgrade from CSR11.5 to CSR 12.5 and in conjunction with this upgrade, moving to a new domain and active directory. With these changes, I decided to assist by ensuring they were compliant with Microsoft's recommendation that secure ldap is used. 
---
While working with a client this week, I encountered an undocumented bug with secure LDAP authentication. My client was doing an upgrade from CSR11.5 to CSR 12.5 and in conjunction with this upgrade, moving to a new domain and active directory. With these changes, I decided to assist by ensuring they were compliant with Microsoft's recommendation that secure ldap is used. 

To begin with, I followed my usual steps that I've documented previously on my [blog][1] without any issue. I was able to get the AD certs imported into my UC applications and successfully sync the directory. As this was the first sync, it was extremely easy to tell it was working. Following this, I tested a couple things like logging into cucm as my own user account and logging into the self care portal. These logins were successful. 

My next steps were to duplicate issues in Unity Connection. This is where the odd behavior began. I could log into things like cobras and the data dump tool as myself, but I couldn't log into CiscoPCA. Later, reports trickled in that jabber users, both on-prem and via MRA, had unreliable access to visual voicemail. Since this was a new active directory, I assumed this was due to a lack of password sync between the old and new domain and just part of the growing pains. 

Fast forward a little and now reports of users trying to login to jabber on-prem are all failing after the cut. In my testing, I disabled secure ldap authentication, and jabber users could all login. I tested the same for CUC and found the same result. I started taking traces on both systems during these failed logins and found this.

```
2021-04-23 17:04:46,507 DEBUG [http-nio-81-exec-17] impl.LDAPHostnameVerifier - check : subjectAlts = [*myclient.org, myclient.org]
2021-04-23 17:04:46,507 ERROR [http-nio-81-exec-17] impl.AuthenticationLDAP - verifyHostName:Exception.javax.net.ssl.SSLPeerUnverifiedException: hostname of the server 'DC2.myclient.org' does not match the hostname in the server's certificate.

2021-04-23 17:04:46,507 DEBUG [http-nio-81-exec-17] impl.AuthenticationLDAP - value of hostnameverifiedfalse
2021-04-23 17:04:46,507 INFO  [http-nio-81-exec-17] impl.AuthenticationLDAP - verifyHostName: Closing LDAP socket

2021-04-23 17:04:46,540 DEBUG [http-nio-81-exec-17] impl.LDAPHostnameVerifier - check : subjectAlts = [*myclient.org, myclient.org]
2021-04-23 17:04:46,540 ERROR [http-nio-81-exec-17] impl.AuthenticationLDAP - verifyHostName:Exception.javax.net.ssl.SSLPeerUnverifiedException: hostname of the server 'DC3.myclient.org' does not match the hostname in the server's certificate.

2021-04-23 17:04:46,541 DEBUG [http-nio-81-exec-17] impl.AuthenticationLDAP - value of hostnameverifiedfalse
2021-04-23 17:04:46,541 INFO  [http-nio-81-exec-17] impl.AuthenticationLDAP - verifyHostName: Closing LDAP socket
```

As you can see, Active Directory is leveraging a wildcard certificate, which is pretty common. However, even though CUCM logins worked fine with this, CUC and IM&P gave these errors, implying that the cert didn't match. A couple options exist to resolve this (I labbed both of these) I could carry 2 intermediate certs, one for the internal systems using a wildcard (the existing one in this case) and then make a new intermediate JUST for the UC applications with specific names. This worked fine in testing and caused minimal headaches. However, getting a new intermediate cert from a client isn't always easy. The other option was to simply change the FQDNs in LDAP authentication to the ip address and then ssh into every CUCM/IM&P/CUC node and type 

`utils ldap config IPAddr`

This resolved the issues for visual voicemail, jabber logins, cisco pca and anything else I may not have found. 

I tried to get Cisco to open a bug on this, but they basically wanted me to break the client's environment so they could collect more logs which wasn't an option. I hope they do the right thing and get a bug open to allow others who don't read my blog to find the issue and solution, but who knows if that will happen. 

A small note, this affects ONLY the ldap authentication portion, not the ldap directory portion. Directory sync worked fine across the board, only authentication failed. I hope this helps some of you!

 [1]: https://blog.longoconsulting.us/2020/02/swap-uc-apps-to-ldaps/