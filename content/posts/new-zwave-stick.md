---
title: New zwave stick
author: driz
date: 2024-03-15T20:30:15.000Z
aliases: /2024/03/new-zwave-stick/
tags: ["docker"]
categories: ["docker","virt"]
summary: As a long-time zwave fan and with most of my home security and other items leveraging zwave, keeping my zwave network optimal is critical for family satisfaction. I was running into an issue where a couple of my further away devices were dying a bit quicker than anticipated and I wanted to get to the bottom of it. Initially, I added some mains powered devices, which also act as routers, to try to fill any potential dead spots, but after 6 months, this didn't help. I finally decided to check firmwares. I was on 7.17.2 and the current version is 7.18.3, so I started looking at the [changelogs/release notes that silicon labs published](https://www.silabs.com/documents/public/release-notes/SRN14889-7.18.3.0.pdf). I really didn't see anything outstanding, but some enhancements to wake-up intervals got added, and potentially that could save battery life.
---

As a long-time zwave fan and with most of my home security and other items leveraging zwave, keeping my zwave network optimal is critical for family satisfaction. I was running into an issue where a couple of my further away devices were dying a bit quicker than anticipated and I wanted to get to the bottom of it. Initially, I added some mains powered devices, which also act as routers, to try to fill any potential dead spots, but after 6 months, this didn't help. I finally decided to check firmwares. I was on 7.17.2 and the current version is 7.18.3, so I started looking at the [changelogs/release notes that silicon labs published](https://www.silabs.com/documents/public/release-notes/SRN14889-7.18.3.0.pdf). I really didn't see anything outstanding, but some enhancements to wake-up intervals got added, and potentially that could save battery life.

So I pulled down the firmware and load it in, error. I went through the arduous process of setting up a windows vm, installing the pc controller, and doing the update following [zooz' process](https://www.support.getzooz.com/kb/article/931-how-to-perform-an-ota-firmware-update-on-your-zst10-700-z-wave-stick/), this failed with the same error. I opened a support ticket with Zooz, if you haven't dealt with them, they are stellar. We had some back and forth on the actual firmware file used, the version of the silicon labs pc controller, and even a picture of the chipset on my stick. They had me open the pc controller again and do a reset on the stick, before doing this, I performed an NVM backup in zwvaejsui so I could easily restore after. After this, they requested that I send them the full firmware version as reported by my controller software. Here are the images sent.
![zwave version from zwavejsui](</images/new-zwave-stick/old zwave stick.png>)
{{< lightgallery assets="new-zwave-stick/zwave chipset.jpg" thumbsize="300x300" >}}

At this point, they responded with the following
`We've seen this error a few times and can offer the following to resolve the issue:1
* We can initiate warranty service for your ZST10 700. Since you're in the US, you can send the stick back to us and we can reset the stick with a special tool in-house (which unfortunately can't be done remotely).
* We can send you an order invoice for a new ZST10 700 stick and have our fulfillment center update it to the latest firmware prior to shipping. Then the old ZST10 700 stick could be sent back to refund on the new order. 

Since so much of my home depends on the zwave network, I selected option 2. They shipped it out USPS, so it took a couple days and inevitably got delivered to the wrong address, but the recipient kindly brought it to me. After receipt, I did a reset of the unit with the silabs pc controller (can never be too safe), took another nvm backup of my existing unit, stopped the zwavejsui container, plugged the new module in, and started the container. As expected I was presented with a pretty empty display.
{{< lightgallery assets="new-zwave-stick/new zwave stick.png" thumbsize="300x300" >}}
As you can see, there is only the single device, all my other devices are missing, but the firmware is 7.18.3, as expected. At this point, it's time to attempt the nvm restore process.
{{< lightgallery assets="new-zwave-stick/1/**.png" thumbsize="300x300" altslice=2 >}}
After this process, I watched the console to ensure things were going as expected, this is what I saw. 
```Shell
2024-03-15 20:15:39.873 INFO Z-WAVE: Calling api restoreNVM with args: [
  <Buffer 01 00 9a b2 01 00 00 d0 fe ff ff 0f ff ff ff ff 2a 58 ff ff 0a 33 00 a8 00 00 00 ff f3 33 00 88 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 ... 49102 more bytes>,
  null,
  [length]: 2
]
INFO Z-WAVE: Controller status: Convert NVM progress: 0%
INFO Z-WAVE: Controller status: Convert NVM progress: 1%
<snip>
INFO Z-WAVE: Controller status: Convert NVM progress: 99%
INFO Z-WAVE: Controller status: Convert NVM progress: 100%
INFO Z-WAVE: Controller status: Restore NVM progress: 0%
INFO Z-WAVE: Controller status: Restore NVM progress: 1%
<snip>
INFO Z-WAVE: Controller status: Restore NVM progress: 99%
INFO Z-WAVE: Controller status: Restore NVM progress: 100%
INFO Z-WAVE: Success zwave api call restoreNVM undefined
INFO Z-WAVE: Controller status: Driver: Applying the NVM backup requires a driver restart! (ZW0100)
INFO Z-WAVE: Restarting client in 1 seconds, retry 1
INFO GATEWAY: Driver is CLOSED
INFO Z-WAVE-SERVER: Client disconnected
INFO Z-WAVE-SERVER: Server closed
INFO Z-WAVE: Client closed
INFO GATEWAY: Driver is CLOSED
INFO Z-WAVE: Connecting to /dev/ttyACM0
INFO Z-WAVE: Setting user callbacks
Logging to file: /usr/src/app/store/logs/zwavejs_2024-03-15.log
INFO Z-WAVE: Zwavejs usage statistics ENABLED
WARN Z-WAVE: Zwavejs driver is not ready yet, statistics will be enabled on driver initialization
INFO GATEWAY: Driver is READY
INFO Z-WAVE: Z-Wave driver is ready
INFO Z-WAVE: Controller status: Driver ready
```
It was pretty clear that the process was successful. I just needed to wait while the stick rebooted, which takes only seconds. I didn't even need to click refresh on the page, I immediately saw the devices show up.
{{< lightgallery assets="new-zwave-stick/after restore.png" thumbsize="300x300" >}}

I performed a couple quick tests to ensure things were behaving as expected, they were, and so I wrote this article. :)  Hopefully this information is useful to someone out there, good luck!