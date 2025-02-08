---
title: 'Unifi Captive Portal'
author: driz
date: '2024-06-24T14:13:55Z'
tags: ["docker"]
categories: ["docker","virt"]

summary: I learned a couple days ago that my oldest daughter has been giving out the family SSID password to kids' friends. I had to (re)explain why this is not allowed and why we have a GUEST network and SSID for GUESTS. 
---

  A few nights ago, my middle daughter had a friend spending the night. She came in to see me and asked if I could log the friend's iPAD into the wifi, I asked her to hold on a few minutes. As I am walking to see her, I see my eldest with a QR code telling her to scan it. Alarms went off immediatley in my head and I asked what she was doing, she said giving her the wifi password and I look and it's the internal family SSID, with access to all of our resources. I told her we would discuss this, grab the iPad and forget the family SSID. I then connect her to the guest network. At this point, I notice that the captive portal didn't show up AND there was no password set. I quickly tossed on a password, reconnected our guest and she was good to go. 

  Moving on from this, I decided to look at what I was thinking and why it was completely open. We don't have a lot of guests, so it's not something I look into very often. Anyway, I log into unifi (I only have Access points, I despise unifi outside of wireless) and check the portal. The setup is pretty basic and I had completed the basics a while back so it was mostly ready. I decided to add an authentication method using a password, though there are other options as well. ![auth methods](</images/unifi-captive-portal/auth-methods.png>) Following this, I went to the actual settings. To start, I had show landing page, https redirection support, and a pre-authorization allowance for my DNS server ip (all dns traffic that attempts to bypass my server is redirected to my server, so the IP must be accessible). ![settings part1](</images/unifi-captive-portal/settings-1.png>)

  This worked fine and I logged in without a problem. Next I enabled secure portal and domain as shown below ![settings part2](</images/unifi-captive-portal/settings-2.png>) and tried connecting, but I got an untrusted site error when connecting to the SSID. After clicking continue with browser, I got another untrusted site error, then I was able to input the password and login. I decided I wanted to resolve this, for a very smooth guest experience, so I checked the cert being issued. It was the default self-signed unifi cert, implying that inside the container was a keystore containing this self-signed cert.  

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

You might be thinking now, well, LE certs expire every 3 months, what then? You're right, the cert will become untrusted every 3 months as it will expire. I will handle this by detecting when a change to the certificates in SWAG occurs, with inspiration from my team-mate [Quietsy's blog article](https://virtualize.link/simplelogin/). To get into specifically what I did, I start with the same renewal-hook quietsy used. My file-name is 20-unifi.sh and the file is stored in `<path to swag>/etc/letsencrypt/renewal-hooks/post` ensure you use with-contenv to pull in the environment settings from the container, though this likely would not cause an issue here if you forget.
```bash{linenos=true}
#!/usr/bin/with-contenv bash

touch /config/etc/letsencrypt/unifi_renew
```
What this does, is any time certbot renews the certs or generates new certs, for whatever reason, this script will run after that completes, so we simply create an empty file. This empty file is the key to determining a change was made.

Now, there are a couple ways to do the next bit, you could add universal-cron to the unifi-network-application image or just do this part on the host and hand it off, i chose to do it on the host. Below is the host-script.
```bash{linenos=true}
#!/bin/bash

set -e
RENEW_PATH=/srv/dev-disk-by-label-Critical/Homes/docker/.config/swag/etc/letsencrypt/unifi_renew

if [ -f ${RENEW_PATH} ]; then
  /usr/bin/curl -H "Authorization: Bearer <redacted>" -Ls -H "Title: Unifi" -d "Attempting to renew guest portal certs" https://ntfy.deez.nuts/alerts
  docker exec -it unifi-network-application bash -c '/config/renew_unifi.sh; exit $?'
  errVal=$?
  if [ ${errVal} -ne 0 ]; then
    /usr/bin/curl -H "Authorization: Bearer <redacted>" -Ls -H "Title: Unifi" -d "Certificate replacement for guest portal failed" https://ntfy.deez.nuts/alerts
  else
    rm -f ${RENEW_PATH}
    /usr/bin/curl -H "Authorization: Bearer <redacted>" -Ls -H "Title: Unifi" -d "Certificate replacement for guest portal successfully completed" https://ntfy.deez.nuts/alerts
  fi
fi
```
We'll break this down, first bear in mind, this is running from the host, not inside the container. We set our renew_path, which should look familiar as it's the file created by the post renewal-hook in SWAG. If that file exists, the script proceeds, I have this on a daily cron because swag renews before the final day meaning I have time.
If the file DOES exist, I first send a ntfy message indicating that the process will begin, then I launch a script within the container. The exit $? ensures I get the exit code from that script running on the host side, if it's a 0 we know it was successful, if it's anything other than 0, there was some kind of error.
We then parse that error code, if it's not a 0, we send a ntfy message letting me know it failed, if it is a 0, we delete the post renewal_hook created file (to avoid superfluously running the script) and then send a ntfy message letting me know it was successful.
So, now we need to see what the script inside the container does

```bash{linenos=true}
set -e
  s6-svc -wd -d /run/service/svc-unifi-network-application
  cd data && cp keystore keystore.bak
  keytool -delete -alias unifi -keystore keystore -deststorepass aircontrolenterprise
  openssl pkcs12 -export -in "/swag-ssl/letsencrypt/live/deez.nuts/fullchain.pem" -inkey "/swag-ssl/letsencrypt/live/deez.nuts/privkey.pem" -out "tempfile" -passout pass:aircontrolenterprise -name unifi
  keytool -importkeystore -srckeystore tempfile -srcstoretype PKCS12 -srcstorepass aircontrolenterprise -destkeystore keystore -deststorepass aircontrolenterprise -destkeypass aircontrolenterprise -alias unifi -trustcacerts
  rm tempfile
  s6-svc -wu -u /run/service/svc-unifi-network-application
```
We'll break this down, first we want to stop the process without stopping the container, this is because in my case, I do not have keystore or openssl installed, so I use them within the container. [-wd -d](https://skarnet.org/software/s6/s6-svc.html) tells s6 to shutdown the service and wait until it's fully shut down, this ensures you don't start making your changes while it's still running, which could potentially lead to corruption.

`s6-svc -wd -d /run/service/svc-unifi-network-application`

After this, I cd into the data directory and backup the keystore `cd data && cp keystore keystore.bak`

Now, we need to do the actual work, we will delete the unifi key from the keystore (which leaves it empty) and use the known (universal) password of `aircontrolenterprise`

```bash
keytool -delete -alias unifi -keystore keystore -deststorepass aircontrolenterprise
```

Now, we need to prep our new certs SWAG has to go into the keystore, using openssl, we create a tempfile that has the fullchain (full trust path and the issued cert) and the private key. You can see we use the same alias and password. Also bear in mind, we are mounting the SWAG ssl directory into our container as /swag-ssl, the host side of this is `<path to swag>/etc/`

```bash
openssl pkcs12 -export -in "/swag-ssl/letsencrypt/live/deez.nuts/fullchain.pem" -inkey "/swag-ssl/letsencrypt/live/deez.nuts/privkey.pem" -out "tempfile" -passout pass:aircontrolenterprise -name unifi
```

Next, we will take the tempfile we created and import it into the keystore. Note we need to use PKCS12 type, the same password, the same alias, and trust the ca certs.

```bash
keytool -importkeystore -srckeystore tempfile -srcstoretype PKCS12 -srcstorepass aircontrolenterprise -destkeystore keystore -deststorepass aircontrolenterprise -destkeypass aircontrolenterprise -alias unifi -trustcacerts
```

Finally, we remove our tempfile we created, since we have no use for it any longer, and then bring back up the service

`rm tempfile` and `s6-svc -wu -u /run/service/svc-unifi-network-application`

Much like when we took the service down, [-wu -u](https://skarnet.org/software/s6/s6-svc.html) will wait until the service is fully up and start the process of bringing it up.

Once this all happens, the unifi process starts up and everything should be good. An already connected user will remain connected, but new guest users may have their ability to connect interrupted during this time which lasts for about 60 seconds for me.