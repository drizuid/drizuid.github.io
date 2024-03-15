---
title: 'SRST-CME -> CME Nightmare'
author: Will
type: posts
date: 2017-02-13T21:12:00+00:00
slug: srst-cme-cme-nightmare
aliases: /2017/02/srst-cme-cme-nightmare/
categories:
  - Cisco Unified Communications
  - route/switch
  - voice
summary: |
  Today, I was working with a client at a remote location. Initially, we prepped a Cisco 4331 to be installed as an SRST-CME device. This particular site has a PRI and a relatively unstable WAN connection. I prepped this router in a different location as we expected no WAN connection once it was installed.

  As it turned out, the PRI got turned up much earlier than expected, so now we needed to get the phones registered to the router. Well, as some of you may know, you can't configure a router for SRST-CME AND CME at the same time. Thus begins the process of removing the SRST-CME config and prepping the new config.
---
Today, I was working with a client at a remote location. Initially, we prepped a Cisco 4331 to be installed as an SRST-CME device. This particular site has a PRI and a relatively unstable WAN connection. I prepped this router in a different location as we expected no WAN connection once it was installed.

As it turned out, the PRI got turned up much earlier than expected, so now we needed to get the phones registered to the router. Well, as some of you may know, you can't configure a router for SRST-CME AND CME at the same time. Thus begins the process of removing the SRST-CME config and prepping the new config.

As I was preparing this, I was notified that rather than 7841 phones, the client would be using 8841 phones! In my head, I thought "no biggie same shit," right? WRONG! As it turned out the 8841 didn't exist as a device type for the version of IOS-XE on my 4331. Like any decent voice guy, I set it up using fast-track.

```Shell
voice register pool-type 8841
xml-config maxnumXalls 6
xml-config busyTrigger 2
phoneload-support
num-lines 6
description Cisco IP Phone 8841
reference-pooltype 8961
```

Well, phones started trying to register, and failing.. there's not a whole lot to a CME config. Build the line, build the device, ezpz.

```Shell
voice register dn  1
 number 5555
 name User1-5555
 label User1-5555
```

```Shell
voice register pool  1
 busy-trigger-per-button 2
 id mac aaaa.bbbb.cccc
 type 8841
 number 1 dn 1
 dtmf-relay rtp-nte
 codec g711ulaw
```

I couldn't figure out why it wasn't registering, I had the xml files needed, everything was there… Finally, in a moment of weakness, I contacted Cisco TAC. I let the dude know what was up and he very quickly said "Oh, you're using fast track config, you need authentication!" WHAT!? We added in the following

```Shell
voice register global
authenticate register
authenticate realm all
!
voice register pool 1
username 5555 password cisco
```

I jumped on to the switch, shut/no shut the phone and boom, registered. Well shit. So I ask, why is this optional setting required? As it turns out, when you use fast track configuration, CME sees the phone as a third party device and thus, requires authentication. Well, I don't want to authenticate, why is the 8800 series line not supported in CME10.5 but 7800 series is? TAC had no clue and suggested upgrading.

Adventure part2  
OK, ISR 4331, I can upgrade to CME 11.0, 11.5, or 11.6. As you can see on this link, we have specific IOS requirements to enabled specific CME versions.&nbsp;[CME Compatibility Matrix](http://web.archive.org/web/20190907085127/http://www.cisco.com/c/en/us/td/docs/voice_ip_comm/cucme/requirements/guide/33matrix.html) Well, after perusing cisco's IOS options for an ISR 4331, I had 2 options, stay where I was on CME 10.5 or upgrade to a non-cisco recommended code base to get 11.5. Obviously, we're upgrading.

I pull the file down, and scp it over to the router

```Shell
domain-name blah.com
crypto key gen rsa mod 1024 gen
ip ssh timeo 3
ip ssh ver 2
ip scp server enable
```

Because I'm a pro, I fucked up the upgrade the first time by typing
 
`boot system flash:.bin`

It failed and told me flash didn't exist and booted back to the original. Well, Cisco just had to change flash to bootflash for.. reasons? fix my command and reboot. I see it come up and surprisingly, attempt to upgrade rommon (I don't think I've even thought of rommon since 1998) and, as you can guess, it failed to upgrade rommon. Thankfully, the new image still booted! Then comes the warning, which my client reads and says "uh… is that bad"

> Detected old ROMMON version 15.4(3r)S5, upgrade required               
> File: /firmware/host/firmware.pkg does not exist ROMMON upgrade failed.    
> Proceeding to boot. A manual ROMMON upgrade is required for the system to function correctly.

uhhhh looks bad to me! but hey, how hard can upgrading rommon be? I'm pretty sure I had to do it on the original CCNA academy I attended in High school. So I pull the rommon down, scp is on to the router and boot into rommon (after myself and a ccie I work with try to remember how the hell you get into rommon) OH YAH the break sequence when you see #! boom, movin now. we get into rommon and `confreg 0x0` so it won't boot into IOS and reset. Now, myself and my ccie colleague are just staring at the screen… Finally, we ask the all-knowing google. first result, we got the command `upgrade rom-monitor filename bootflash:.bin all` command upgrade not found.. The CCIE suggests it's probably a hidden command, sounds legit, we bang away, try a usb device, do some more searches for about 30 minutes. Finally, I say, let's try doing this from in IOS. `confreg 0x2102`

We boot up, and of course as soon as we're in IOS, the command is available! Well, lesson learned, you upgrade rommon from within IOS. The upgrade took about 15 minutes and finishes, I reboot for the rommon upgrade to take effect and we no longer see the error regarding rommon.

Success! We boot up and it tells me that my 8841 fast-track config isn't needed because 8841 is built-in! Great!! As it removed my fast track config, I did have to re-add `type 8841` to each device, but not a big deal. This was a 10 hour day between waiting on the telco to show up to turn up the PRI, getting the cable vendor in and getting everything built in UCM, Unity (for once the WAN connection is up) and of course building the CME config on the router. This site previously used layer3 routers so I also reconfigured the subinterfaces to be on the router and use your typical router-on-a-stick configuration. We also discussed some redundancy options that we can implement later. All in all, a painful but educational day.
