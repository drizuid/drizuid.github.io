---
title: Using SQL to clean up Call Manager pt3
author: Will
type: post
date: 2017-05-18T20:23:00+00:00
url: /2017/05/using-sql-to-clean-up-call-manager-pt3/
categories:
  - Cisco Unified Communications
  - informix
  - sql
  - voice

---
See Parts&nbsp;[1][1]&nbsp;and&nbsp;[2][2]

If you've been following me in this 3 part series, you know we started off with around 700 dependencies on a CSS that no longer fits our standard. It was in use by various things and we leveraged SQL to quickly, efficiently, and safely remove it from use. When we finished part 2, the only things still referencing our css were directory numbers. Well, we actually have 2 CSSs we're going to clean up today.

As usual, we'll begin with a count.  
![1](/posts/using-sql-to-clean-up-call-manager-pt3/1.PNG)

![5](/posts/using-sql-to-clean-up-call-manager-pt3/5.PNG)

To ensure we don't mess things up, we will verify the pkid of the css in question.  
![2](https://blog.longoconsulting.us/images/sql_device/2.PNG)

![3](/posts/using-sql-to-clean-up-call-manager-pt3/3.PNG)

Let's check where these are being used so we can check our data dictionary for the names to use.  
![2](/posts/using-sql-to-clean-up-call-manager-pt3/2.PNG)
I spot checked some of the records and nearly all looked the same, so our course of action would be to assume MOST look like the above, and address this first, then attack the outliers. In this case, I want to set the CSS to NONE, but&nbsp;**ONLY** if the CSS in use is what I expect.![4](/posts/using-sql-to-clean-up-call-manager-pt3/4.png) 
We affected 12 records with this which is pretty nice. considering our highest record count above was a 12. This means we probably have a couple outliers who are perhaps missing one or two of the fields in our sql where clause.

To catch the remaining PRO-BHM-CfwdUnreg-CSS guys, I ran this and happened to get the rest.<figure class="wp-block-image">

![6](/posts/using-sql-to-clean-up-call-manager-pt3/6.png)

I forgot to take screenshots, but the remaining PRO-Device-CSS entries were because the forward on CTI failure CSS was already set to NONE and my where clause required it to have the css in it. I modified my query and captured the remaining records. It took about 5 minutes to prep and run the query, including research. So, in all, to remove 700 dependencies on a CSS where BAT couldn't handle it, we took about 15 minutes while capturing screens. In a real world setting where you didn't care about blogging, this is probably about 10 minutes of work in SQL and extremely safe to do. I've now deleted both CSSs we referenced above and this mini-series is complete.

Thanks for reading!

Here is the data dictionary for [10.5](https://developer.cisco.com/media/UCM10.5DataDictionary/UCM10.5DataDictionary.htm)

I used the numplan table today.

 [1]: /2017/04/using-sql-to-clean-up-call-manager-pt1/
 [2]: /2017/05/using-sql-to-clean-up-call-manager-pt2/
 [3]: /posts/using-sql-to-clean-up-call-manager-pt3/5.PNG
 [4]: /posts/using-sql-to-clean-up-call-manager-pt3/6.png