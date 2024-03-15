---
title: Using SQL to clean up Call Manager pt2
author: Will
type: posts
date: 2017-05-02T20:22:00+00:00
slug: using-sql-to-clean-up-call-manager-pt2
aliases: /2017/05/using-sql-to-clean-up-call-manager-pt2/
series: ['Using SQL to cleanup CUCM']
categories:
  - Cisco Unified Communications
  - informix
  - sql
  - voice
summary: |
  Previously we had around 700 dependencies on a CSS that no longer fits our standard. This CSS was in use by lines, devices, and users throughout the system. In part 1 we removed this css from the users, now we need to do the same for the devices.

  As before, we'll start with a count.  
  ![count](/images/using-sql-to-clean-up-call-manager-pt2/count.PNG)
---

See Parts&nbsp;[1][1]&nbsp;and&nbsp;[3][2]

Previously we had around 700 dependencies on a CSS that no longer fits our standard. This CSS was in use by lines, devices, and users throughout the system. In part 1 we removed this css from the users, now we need to do the same for the devices.

As before, we'll start with a count.  
![count](/images/using-sql-to-clean-up-call-manager-pt2/count.PNG)

Next, we will check for the pkid of the callingsearchspace to be removed  
![](/images/using-sql-to-clean-up-call-manager-pt2/2.PNG)
Now, let's check one of our devices from the 22 and see WHERE the css is being used.  
![](/images/using-sql-to-clean-up-call-manager-pt2/3.PNG) 
We see our selected device is using it as the rerouting CSS, we will want to see if it's being used in the same spot once we know the column name. We'll need to check our datadictionary to see what the column name for the rerouting calling search space is on a device. Since we're working with a device, we know to look under the device table.  
![](/images/using-sql-to-clean-up-call-manager-pt2/4a.PNG) 
We found something promising, let's verify this is the correct column.  
![](/images/using-sql-to-clean-up-call-manager-pt2/5a.PNG) 
Luckily, it is being used on all 22 devices in the same place, this makes the clean up extremely easy. We'll grab the PKID for the CSS we are changing too and then apply to a single phone.  
![](/images/using-sql-to-clean-up-call-manager-pt2/6.PNG) 
and applying  
![](/images/using-sql-to-clean-up-call-manager-pt2/7.PNG) 
We see that, as expected, our change applied to one device, let's verify our work now.  
![](/images/using-sql-to-clean-up-call-manager-pt2/8.PNG) 
Look's good! Let's apply to the rest of the 21 remaining devices now!  
![](/images/using-sql-to-clean-up-call-manager-pt2/9.PNG) 
We applied the change to 21 devices, as expected, let's refresh our dependency record window.  
![](/images/using-sql-to-clean-up-call-manager-pt2/10.PNG) 
![](/images/using-sql-to-clean-up-call-manager-pt2/11.PNG) 
All of our devices are now changed, as you can see both via GUI and SQL.

Since we had the PKIDs from our previous entry, this took approximately 1.5 minutes to get fixed. Doing this manually would have taken at least 1minute per device manually or around 10 minutes to export the phones, modify the bat file and reimport. SQL is dangerous, certainly, but by taking small steps, we can do it safely and quickly.

Here is the data dictionary for [10.5](https://developer.cisco.com/media/UCM10.5DataDictionary/UCM10.5DataDictionary.htm)

I used the device and callingsearchspace tables today.

 [1]: /2017/04/using-sql-to-clean-up-call-manager-pt1/
 [2]: /2017/05/using-sql-to-clean-up-call-manager-pt3/