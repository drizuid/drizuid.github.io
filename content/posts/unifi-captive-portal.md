---
title: 'Unifi Captive Portal'
author: driz
date: '2024-06-24T14:13:55Z'
tags: ["docker"]
categories: ["docker","virt"]

summary: I learned a couple days ago that my oldest daughter has been giving out the family SSID password to kids' friends. I had to (re)explain why this is not allowed and why we have a GUEST network and SSID for GUESTS. 
---

  A few nights ago, my middle daughter had a friend spending the night. She came in to see me and asked if I could log the friend's iPAD into the wifi, I asked her to hold on a few minutes. As I am walking to see her, I see my eldest with a QR code telling her to scan it. Alarms went off immediatley in my head and I asked what she was doing, she said giving her the wifi password and I look and it's the internal family SSID, with access to all of our resources. I told her we would discuss this, grab the iPad and forget the family SSID. I then connect her to the guest network. At this point, I notice that the captive portal didn't show up AND there was no password set. I quickly tossed on a password, reconnected our guest and she was good to go. 

  Moving on from this, I decided to look at what I was thinking and why it was completely open. We don't have a lot of guests, so it's not something I look into very often. Anyway, I log into unifi (I only have Access points, I despise unifi outside of wireless) and check the portal. The setup is pretty basic and I had completed the basics a while back so it was mostly ready. I decided to add an authentication method using a password, though there are ![other options](</images/unifi-captive-portal/auth-methods.png>). Following this, I went to the actual settings. To start, I had show landing page, https redirection support, and a pre-authorization allowance for my DNS server ip (all dns traffic that attempts to bypass my server is redirected to my server, so the IP must be accessible). ![settings part1](</images/unifi-captive-portal/settings-1.png>)

  This worked fine and I logged in without a problem. Next I enabled secure portal and domain ![settings part2](</images/unifi-captive-portal/settings-2.png>) and tried connecting, but I got an untrusted site error when connecting to the SSID. After clicking continue with browser, I got another untrusted site error, then I was able to input the password and login. I decided I wanted to resolve this, for a very smooth guest experience, so I checked the cert being issued. It was the default self-signed unifi cert, implying that inside the container was a keystore containing this self-signed cert.  

  I shelled into the container and started my search, it was pretty simple, when you bash in, you are in `/usr/lib/unifi` already, within this location is a `data` folder. Inside `data` is a file named `keystore`. Using keytool, I checked the contents and found the unifi self-signed cert. With this found, I knew my next steps are to map my swag certs into the unifi-network-application container, delete the self-signed cert in keystore, and then create my own certs inside the keystore. Here are the steps I followed

  1) Ensure SWAG certs are mapped and available, mine are in /swag-ssl
  2) s6-svc -d /run/service/svc-unifi-network-application
3) `cd data && cp keystore keystore.bak` we backup the keystore file, just in case :)
4) `keytool -list -keystore keystore` we list what's in the keystore, I noted the fingerprint and date to compare once finished
	Enter keystore password:
	Keystore type: PKCS12
	Keystore provider: SUN

	Your keystore contains 1 entry

	unifi, Sep 6, 2023, PrivateKeyEntry,
	Certificate fingerprint (SHA-256): CF:24:96:BD:50:7D:D3:30:DA:DC:AA:FA:CD:9A:5E:20:7B:71:62:C7:E1:7B:73:A1:50:D0:C8:B6:5A:79:A5:45
5) `keytool -delete -alias unifi -keystore keystore -deststorepass aircontrolenterprise` we delete the cert inside the keystore (it's still in our backup file just in case)
6) you can do this in the swag-ssl folder or just symlink the files into data i symlinked -- `ln -s /swag-ssl/letsencrypt/live/<mydomain>/*.pem ./` I symlinked the certs into the data directory, it's messy, but simpler when blogging it.
7) `openssl pkcs12 -export -in "fullchain.pem" -inkey "privkey.pem" -out "tempfile" -passout pass:aircontrolenterprise -name unifi` here we need to take the swag PEM files and create a pkcs12 file using the standard unifi password. Fullchain.pem contains the signed cert AND the CA cert, then we just bake in the privatekey.
8) `keytool -importkeystore -srckeystore tempfile -srcstoretype PKCS12 -srcstorepass aircontrolenterprise -destkeystore keystore -deststorepass aircontrolenterprise -destkeypass aircontrolenterprise -alias unifi -trustcacerts` this imports our pkcs12 file we created into the keystore
	Importing keystore tempfile to keystore...
9) `keytool -list -keystore keystore` now we confirm it's added, we should see a new date and a new fingerprint
	Enter keystore password:
	Keystore type: PKCS12
	Keystore provider: SUN

	Your keystore contains 1 entry

	unifi, Jun 24, 2024, PrivateKeyEntry,
	Certificate fingerprint (SHA-256): 7C:3F:C4:CC:8F:04:35:A4:B3:6F:00:54:97:DB:DC:FE:92:05:2B:39:E2:5A:92:8E:5E:88:81:41:D9:01:17:B0
10) `rm tempfile` clean up our tempfile
11) `exit` break out of the container
12) `docker restart unifi-network-application` restart to let the service come back up

At this point, it should be ready. I connected to the guest wifi and got a trusted dialog with the imagery i set in the landing setup page. I input my password and I was connected. There was no attempt to open a new page and no trust issues.

You might be thinking now, well, LE certs expire every 3 months, what then? You're right, the cert will become untrusted every 3 months as it will expire. The best way to handle this, in my opinion, is a [custom script](https://www.linuxserver.io/blog/2019-09-14-customizing-our-containers) that will update the keystore, this would require a container restart, but I'm still pondering other ideas right now. If you have any ideas, feel free to ping me on the linuxserver.io discord and share them :)