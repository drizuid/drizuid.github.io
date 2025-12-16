---
title: 'Multicast Video Across Vlans'
author: driz
date: '2024-03-18T11:39:39Z'
tags: ["docker", "router"]
categories: ["docker", "Linux"]
summary: As some of you may know from prior posts, I have a number of external security cameras (and internal) that show up on monitors throughout the house 24/7. To keep things efficient, these camera streams are multicast feeds the monitors subscribe to. Unfortunately, every so often, I need to check the streams on my PC which is in a different VLAN. I was having to access the unicast streams and I wanted to work out getting multicast to work across vlan boundaries in OPNsense. Usually, this would be something IGMP and PIM can handle in my world, but I have no Cisco gear in my network and while pfsense has igmp and pimd, OPNsense lacks pimd, so the research began.
---

As some of you may know from prior posts, I have a number of external security cameras (and internal) that show up on monitors throughout the house 24/7. To keep things efficient, these camera streams are multicast feeds the monitors subscribe to. Unfortunately, every so often, I need to check the streams on my PC which is in a different VLAN. I was having to access the unicast streams and I wanted to work out getting multicast to work across vlan boundaries in OPNsense. Usually, this would be something IGMP and PIM can handle in my world, but I have no Cisco gear in my network and while pfsense has igmp and pimd, OPNsense lacks pimd, so the research began.

As mentioned in previous posts, I use Amcrest cameras in my home. They are dahua knock-offs and work quite well. You can use them FULLY offline, they support onvif, rtsp, msjpeg, etc. When I first started using them, multicast settings appeared right in the UI, but in later firmware versions it was gone. I was initially concerned that the features were removed which would've been quite sad, but I decided to check. Using `http://ip/cgi-bin/configManager.cgi?action=getConfig&name=All` will give you the full config of your camera and yes, you can set the config like this as well. Looking through the config, I quickly found the multicast section `table.All.Multicast.` To ensure I knew what I was doing, I also pulled up a copy of the [Amcrest API docs](https://s3.amazonaws.com/amcrest-files/AMCREST_CGI_SDK_API.pdf), unfortunately, the multicast section is completely missing from the API SDK I found. Without strict guidance on the settings, I made some educated guesses and made a task list.
* Set the multicast IP I want to use
  * do this for the main stream AND the substream
* Set the multicast port I want to use
  * do this for the main stream AND the substream
* Set some things to false that I won't be using such as
  * DHII
  * multicast RTP
* Set multicast TS to true

To accomplish this, I used a bash script which is shown below.
```bash
password=mysecretcameraP455word
i=9
for n in fronty backy drive bdoor proom lroom basement
do
	curl -s --digest -u admin:$password -g "http://cam-$n/cgi-bin/configManager.cgi?action=setConfig&Multicast.TS[1].MulticastAddr=224.1.3.$i"
	curl -s --digest -u admin:$password -g "http://cam-$n/cgi-bin/configManager.cgi?action=setConfig&Multicast.TS[0].MulticastAddr=224.1.2.$i"
	curl -s --digest -u admin:$password -g "http://cam-$n/cgi-bin/configManager.cgi?action=setConfig&Multicast.TS[1].Port=20016"
	curl -s --digest -u admin:$password -g "http://cam-$n/cgi-bin/configManager.cgi?action=setConfig&Multicast.TS[0].Port=20000"
	curl -s --digest -u admin:$password -g "http://cam-$n/cgi-bin/configManager.cgi?action=setConfig&Multicast.DHII[0].Enable=false"
	curl -s --digest -u admin:$password -g "http://cam-$n/cgi-bin/configManager.cgi?action=setConfig&Multicast.DHII[1].Enable=false"
	curl -s --digest -u admin:$password -g "http://cam-$n/cgi-bin/configManager.cgi?action=setConfig&Multicast.RTP[0].Enable=false"
	curl -s --digest -u admin:$password -g "http://cam-$n/cgi-bin/configManager.cgi?action=setConfig&Multicast.RTP[1].Enable=false"
	curl -s --digest -u admin:$password -g "http://cam-$n/cgi-bin/configManager.cgi?action=setConfig&Multicast.RTP[2].Enable=false"
	curl -s --digest -u admin:$password -g "http://cam-$n/cgi-bin/configManager.cgi?action=setConfig&Multicast.TS[0].Enable=true"
	curl -s --digest -u admin:$password -g "http://cam-$n/cgi-bin/configManager.cgi?action=setConfig&Multicast.TS[1].Enable=true"
	curl -s --digest -u admin:$password -g "http://cam-$n/cgi-bin/configManager.cgi?action=setConfig&Multicast.TS[2].Enable=false"
	((i=i-1))
	echo $n
done
```
As you can see from the script, I elected to use 224.1.2.X for main stream, with port 20000 and 224.1.3.X for sub stream, with port 20016. After getting this setup, I tried to play the multicast stream from one of my raspberry PI units using omxplayer. `omxplayer udp://224.1.3.7:20016 --live -n -1` I ran this command and got a full screen stream of the camera! Just to be sure things weren't being squirrely, I ran the same command on another PI and got the same stream. I checked wireshark and I saw the IGMP joins from the two nodes in question which definitively confirmed this was working. However, at this point, the PIs and cameras are on the same vlan, vlan55.

During my wireshark, I also noticed that some of my cameras that were wireless only were in the membership queries, I realized I had run my bash script on every camera. In my case, proom, lroom, and basement are wireless and while my network will support multicast over wireless, it is not something I wanted to do at this time. I simply set Multicast.TS[0] and [1] to false on those cameras and they stopped participating in the multicast process.

Before moving forward, I documented the ip and multicast address per camera in my test suite. 
{{< table >}}
| name       | mcast ip | network ip |
|:----------:|:--------:|:----------:|
|basement    | X.3      |    55.27   |
|lroom       | X.4      |    55.26   |
|proom       | X.5      |    55.25   |
|bdoor       | X.6      |    55.22   |
|drive       | X.7      |    55.20   |
|backy       | X.8      |    55.23   |
|fronty      | X.9      |    55.24   |
{{< /table >}}

At this point, I started researching my options in OPNsense. In my case, I already had a plugin called `UDP Broadcast Relay` that I was using to allow me to cast from my LAN vlan128 to my IoT vlan155. I decided to start here and see if I could make it work. In the configuration of this plugin, you input the relay port, relay interfaces, broadcast addresses, source address, and instance id. For port, I input 20016, my substream multicast port, for relay interfaces, I input LAN, and NoT which are the OPNSense interface names for vlan128 and vlan55, for broadcast addresses, I input 224.1.3.6-9 for the wired cameras. For source address, I found this to be a bit confusing, but for my test case, I selected 1.1.1.2, though at this point, i think I could've left it blank. My instance ID is just 1.
![opnsense udp relay settings](</images/multicast-video-across-vlans/udp relay substream.png>)

With that in place, I decided to check tcpdump and see if anything magical was happening. I made liberal use of `sudo tcpdump -i igc3 -nv '(multicast and not broadcast and not udp port 546 and not udp port 547)'` 
the filter on 546 and 547 is because dhcp6 on the LAN vlan was spamming me to hell. (and i was already being spammed by the multicast). I did this directly on opnsense itself. I saw plenty of traffic on vlan01, which is what opnsense calls vlan55, but nothing on vlan02, which is what opnsense calls vlan128. 
```
21:13:27.452471 IP (tos 0x0, ttl 64, id 35703, offset 0, flags [DF], proto UDP (17), length 1344)
    192.168.55.22.20000 > 224.1.2.6.20000: UDP, length 1316
21:13:27.452570 IP (tos 0x0, ttl 64, id 18984, offset 0, flags [DF], proto UDP (17), length 216)
    192.168.55.22.20000 > 224.1.3.6.20016: UDP, length 188
21:13:27.452652 IP (tos 0x0, ttl 64, id 6946, offset 0, flags [DF], proto UDP (17), length 1344)
    192.168.55.20.20000 > 224.1.2.7.20000: UDP, length 1316
```
I pondered for a moment and my thought was that opnsense blocks inter-vlan traffic by default, so that could be the case here. I headed over to **Firewall>rules>NoT**.
From here, I created 3 rules:
* allow IPv4 IGMP destined to 224.0.0.0/4 on the NoT network
* allow IPv4 PIM destined to 224.0.0.0/4 on the NoT network
* allow IPv4 UDP from my cameras (this is an alias containing the network ip of my cameras) destined to 224.0.0.0/4.
![firewall rules for multicast](</images/multicast-video-across-vlans/firewall rules.png>)
As soon as I clicked save and apply, I saw the traffic hit vlan02.
```
21:13:29.842793 IP (tos 0x4, ttl 64, id 4660, offset 0, flags [none], proto UDP (17), length 216)
    192.168.128.1.20000 > 224.1.3.6.20016: UDP, length 188
21:13:29.848873 IP (tos 0x4, ttl 64, id 4660, offset 0, flags [none], proto UDP (17), length 404)
    192.168.128.1.20000 > 224.1.3.9.20016: UDP, length 376
21:13:29.857901 IP (tos 0x4, ttl 64, id 4660, offset 0, flags [none], proto UDP (17), length 216)
    192.168.128.1.20000 > 224.1.3.8.20016: UDP, length 188
21:13:29.862494 IP (tos 0x4, ttl 64, id 4660, offset 0, flags [none], proto UDP (17), length 404)
    192.168.128.1.20000 > 224.1.3.7.20016: UDP, length 376
21:13:29.872839 IP (tos 0x4, ttl 64, id 4660, offset 0, flags [none], proto UDP (17), length 1344)
    192.168.128.1.20000 > 224.1.3.7.20016: UDP, length 1316
```
Now I noticed that I was missing my mainstream (224.1.2.X) in the tcpdump on vlan02, but I had some in vlan01 (see above), I realized that this was because I forgot to create a relay rule in udp broadcast relay, for that port and those devices. I quickly head over to create that.
![opnsense udp relay settings](</images/multicast-video-across-vlans/udp relay mainstream.png>)
As soon as I finished, I saw the traffic on vlan02 begin, now from both subnets.
```
21:13:37.281885 IP (tos 0x4, ttl 64, id 4660, offset 0, flags [none], proto UDP (17), length 404)
    192.168.128.1.20000 > 224.1.3.7.20016: UDP, length 376
21:13:37.290516 IP (tos 0x4, ttl 64, id 4660, offset 0, flags [none], proto UDP (17), length 216)
    192.168.128.1.20000 > 224.1.3.9.20016: UDP, length 188
21:13:37.306149 IP (tos 0x10, ttl 64, id 4660, offset 0, flags [none], proto UDP (17), length 404)
    192.168.128.1.20000 > 224.1.2.8.20000: UDP, length 376
21:13:37.312912 IP (tos 0x10, ttl 64, id 4660, offset 0, flags [none], proto UDP (17), length 1344)
    192.168.128.1.20000 > 224.1.2.6.20000: UDP, length 1316
21:13:37.312958 IP (tos 0x10, ttl 64, id 4660, offset 0, flags [none], proto UDP (17), length 1344)
    192.168.128.1.20000 > 224.1.2.6.20000: UDP, length 1316
21:13:37.313084 IP (tos 0x10, ttl 64, id 4660, offset 0, flags [none], proto UDP (17), length 1344)
    192.168.128.1.20000 > 224.1.2.6.20000: UDP, length 1316
21:13:37.313106 IP (tos 0x4, ttl 64, id 4660, offset 0, flags [none], proto UDP (17), length 216)
    192.168.128.1.20000 > 224.1.3.6.20016: UDP, length 188
21:13:37.316506 IP (tos 0x10, ttl 64, id 4660, offset 0, flags [none], proto UDP (17), length 1344)
    192.168.128.1.20000 > 224.1.2.8.20000: UDP, length 1316
```

With the cameras now sending out membership queries in their own vlan, which opnsense was relaying over to my LAN vlan, I should be able to send a membership report from my client (aka, subscribe to the multicast feed). I load up VLC and input `udp://@224.1.2.9:20000` for the mainstream of my front yard (fronty) camera and the video immediately popped up.
![multicast stream](</images/multicast-video-across-vlans/multicast camera.png>)

At this point, everything is working as expected. Unfortunately, I noticed some less than ideal things, it's quite spammy. I suspect this is because udp broadcast relay is spamming things out regardless of whether the feed is subscribed to. I'll most likely disable the rules because the efficiency gains I expected are moot with the way the relay seems to be working, but this was a fun exercise either way.

I'll list some other commands of interest and links to things I used in my research. The two commands below will get the config sections for Network and Multicast, respectively.

* `http://<cam>/cgi-bin/configManager.cgi?action=getConfig&name=Network`
* `http://<cam>/cgi-bin/configManager.cgi?action=getConfig&name=Multicast`

AD410 and 110 doorbells also use the same API and can be made fully offline with some effort, for example, to see the full config and then set the NTP address to something specific you can use the following commands respectively.

* `http://<AD110-IP>/cgi-bin/Config.backup?action=All`
* `http://<AD110-IP>/cgi-bin/configManager.cgi?action=setConfig&NTP.Address=<ip or url to time server>`

Here are some sources I used during research:

* [https://support.amcrest.com/hc/en-us/articles/360018625512-How-To-Setup-Multicast](https://support.amcrest.com/hc/en-us/articles/360018625512-How-To-Setup-Multicast)
* [https://www.reddit.com/r/raspberry_pi/comments/4elhxi/omxplayer_streaming_source_specific_multicast/](https://www.reddit.com/r/raspberry_pi/comments/4elhxi/omxplayer_streaming_source_specific_multicast/)

I did not go into any detail on this, but your switches need to support igmp and it must be enabled on the ports participating. In my case this means the access ports for the cameras themselves, the access ports for the raspberry pi units viewing the streams, and the trunk ports to the core switch and opnsense. From a prior post, here is what one of the security monitors look like at the time I made that post.
{{< lightgallery assets="new-home-new-network/Security camera monitor.jpg" thumbsize="300x300" >}}