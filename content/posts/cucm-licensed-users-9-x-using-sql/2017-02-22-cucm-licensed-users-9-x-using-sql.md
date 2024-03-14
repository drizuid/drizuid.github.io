---
title: CUCM Licensed Users (9.x+) using SQL
author: Will
type: post
date: 2017-02-22T21:19:00+00:00
url: /2017/02/cucm-licensed-users-9-x-using-sql/
categories:
  - Cisco Unified Communications
  - informix
  - sql
  - voice

---
A client was facing some licensing issues with shared devices. After a discussion, we decided to create a local user account for this particular location. Using this local user account, we would assign all generic shared devices (waiting room, lobby, hallway) to this user and save a bunch of enhanced licenses by properly utilizing CUWLs.

I provided some guidance and a local admin began the process of clicking a phone, setting the user, saving and picking the next. Obviously this is pretty painful and slow. I decided to see what we could do to speed the process up.

<!--more-->

It began with a few discovery queries. I knew what user I was assigning, so I needed to find the pkid of that user. Leveraging the CLI of the UCM (thanks William Bell!!) I constructed a query.

`run sql select pkid from EndUser where userid='Location1User'`

which returned

`'2685b6aa-fdd3-413a-16bc-589e768ba4ab'`

Next, I wanted to get a count of devices. I needed some filters on this particular query to make sure

  1. We return only phones
  2. We return only phones at the location in question
  3. We return only phones that are set to anonymous for the licensed user

I utilized this particular query:

`run sql select count(name) from Device where name NOT LIKE 'UCCX%' AND name NOT LIKE 'CER%' AND name NOT LIKE 'ATA%' AND name NOT LIKE 'AN%' AND tkclass=1 AND fkenduser is null AND (fkdevicepool='bb8c3bf4-2ace-dba7-c8b1-32dc509f1479' OR fkdevicepool='1b1b9eb6-7803-11d3-bdf0-00108302ead1'`

The location in question has two devicepools, one for srst and one for non-srst. Obviously, I didn't want to put essential or basic licensed devices into my CUWL group, so i filtered analog and cti route points. This gave me a count of 103 devices which lined up with the manual count the local admin had completed. Now, I could quickly remove count() and create 103 update statements to set the user, with something like `run sql update Device set fkenduser='2685b6aa-fdd3-413a-16bc-589e768ba4ab' where name='SEP94D4692B566E'`, but I like to overthink things.

I instead used the pre-created filter I used to verify count and updated anything matching that.

`run sql update Device set fkenduser='2685b6aa-fdd3-413a-16bc-589e768ba4ab' where name NOT LIKE 'UCCX%' AND name NOT LIKE 'CER%' AND name NOT LIKE 'ATA%' AND name NOT LIKE 'AN%' AND tkclass=1 AND fkenduser is null AND (fkdevicepool='bb8c3bf4-2ace-dba7-c8b1-32dc509f1479' OR fkdevicepool='1b1b9eb6-7803-11d3-bdf0-00108302ead1'`

Knowing I had 103 phones from our previous physical check AND my query, i expect a result of 102 rows updated.

`Rows: 102`

which is exactly what I got. So, we do a quick check now. Before running my sql update, i saw the following on a certain phone (simply the last one in my original scan) (**These pictures were lost and can not be recovered)**

All in all, the process, including the discovery queries took less than 20 minutes (which included typing up this blog post). Time saver? I would say so.

I've been doing some work using SQL in UCM for various clients for some time now, and while risky, careful testing with select statements and a decent understanding of SQL (and cisco's data dictionary) make it quick and safe!