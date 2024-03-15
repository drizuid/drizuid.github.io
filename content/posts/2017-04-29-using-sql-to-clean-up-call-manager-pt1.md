---
title: Using SQL to clean up Call Manager pt1
author: Will
type: posts
date: 2017-04-29T20:20:00+00:00
slug: using-sql-to-clean-up-call-manager-pt1
aliases: /2017/04/using-sql-to-clean-up-call-manager-pt1/
series: ['Using SQL to cleanup CUCM']
categories:
  - Cisco Unified Communications
  - informix
  - sql
  - voice
summary: |
  So, at a client of mine that we will call PRO, we had a Device-CSS which seemed to be the default css for pretty much everything, including presence subscription, even though PRO-Subscribe-CSS exists. Well the PRO-Device-css also used a lot of legacy stuff from the PRI days. I wanted to clean it up and swap things to follow the standards we had implemented, which required naming based on the line of business, location, and use.

  First I modified all the templates to remove the PRO-Device-CSS usage and replace it with the appropriate css.  
  Next I check dependencies, assuming there couldn’t be that many things…

  ![](/images/using-sql-to-clean-up-call-manager-pt1/1.png)
---

See Parts&nbsp;[2][1]&nbsp;and&nbsp;[3][2]

So, at a client of mine that we will call PRO, we had a Device-CSS which seemed to be the default css for pretty much everything, including presence subscription, even though PRO-Subscribe-CSS exists. Well the PRO-Device-css also used a lot of legacy stuff from the PRI days. I wanted to clean it up and swap things to follow the standards we had implemented, which required naming based on the line of business, location, and use.

First I modified all the templates to remove the PRO-Device-CSS usage and replace it with the appropriate css.  
Next I check dependencies, assuming there couldn’t be that many things…

![](/images/using-sql-to-clean-up-call-manager-pt1/1.png)

Well then.. i figured I would start with end users.. simply bulk update users where the subscribe css = "PRO-device-css" and update.. except you can’t.. so I exported all users but was again lacking this particular field in the export data..  
Well, we know all callingsearchspaces are in the callingsearchspace table and all endusers are in the enduser table. I took the first user in the list, "pbryan" and checked him out.

![](/images/using-sql-to-clean-up-call-manager-pt1/2.png) 

well, we need to verify this is the right field, so

![](/images/using-sql-to-clean-up-call-manager-pt1/3.png)

Looks good!  
Now let’s get the pkid for these two CSS  
We already know PRO-Device-CSS is 8a7b049d-12ea-1bbf-1d76-1b09cc922b72 so let’s find out what PRO-Subscribe-CSS is

![](/images/using-sql-to-clean-up-call-manager-pt1/4.png)

Now we can simply do an update on endusers who have the pkid of PRO-Device-CSS as their fkcallingsearchspace_restrict and set it to the PKID of PRO-Subscribe-CSS.  
As usual, we want to limit our test to 1 before blasting all 625. We will use our planned filter and include another clause specifying the userid.

![](/images/using-sql-to-clean-up-call-manager-pt1/5.png)

We see that 1 row was affected, which was our expected result. Let’s check our work

![](/images/using-sql-to-clean-up-call-manager-pt1/6.png)

above pbryan had PRO-Device-CSS now he has PRO-Subscribe-CSS, so our sql statement worked as expected. Let’s remove the userid filter and apply the change to everyone using the incorrect css.

![](/images/using-sql-to-clean-up-call-manager-pt1/8.png)

we see 624 rows affected, which we expect since we had 625 and handled 1 already.

![](/images/using-sql-to-clean-up-call-manager-pt1/7.png)

There’s still a ton of shit to clean up but, considering it took me longer to type this blog entry than to do the work, time well spent.

Here is the data dictionary for [10.5](https://developer.cisco.com/media/UCM10.5DataDictionary/UCM10.5DataDictionary.htm)

I used the enduser and callingsearchspace tables today.

 [1]: /2017/05/using-sql-to-clean-up-call-manager-pt2/
 [2]: /2017/05/using-sql-to-clean-up-call-manager-pt3/