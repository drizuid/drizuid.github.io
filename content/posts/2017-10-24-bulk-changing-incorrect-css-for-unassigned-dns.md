---
title: Bulk changing incorrect css for unassigned DNs
author: Will
type: posts
date: 2017-10-24T20:24:00+00:00
url: /2017/10/bulk-changing-incorrect-css-for-unassigned-dns/
categories:
  - Cisco Unified Communications
  - informix
  - sql
  - voice
summary: Today I was cleaning up some CSSs for a client. I came across a particular css that had been erroneously assigned as the line css for a bunch of unassigned DNs (they were precreated to show they were already in use). Of course, I went to BAT first to see if I could just update the line css of the lines, but I discovered that I couldn't affect the unassigned DNs (even though there is an option for searching unassigned dns…) Anyway, as you can guess, I jumped into SQL to see what I could do.
---
Today I was cleaning up some CSSs for a client. I came across a particular css that had been erroneously assigned as the line css for a bunch of unassigned DNs (they were precreated to show they were already in use). Of course, I went to BAT first to see if I could just update the line css of the lines, but I discovered that I couldn't affect the unassigned DNs (even though there is an option for searching unassigned dns…)

Anyway, as you can guess, I jumped into SQL to see what I could do.

1) Determine the pkid of the erroneous CSS

```
admin:run sql select pkid from callingsearchspace where name="Long-Distance_CSS"
pkid
====================================
77f5f4de-f98c-1a13-728c-402623cb9ca0
```

2) Determine which phones are using the CSS (can compare against dependency records) using a select statement. I already know the css name is "Long-Distance_CSS"

```
admin:run sql select dnorpattern as dn, name as line_css from numplan inner join callingsearchspace on numplan.fkcallingsearchspace_sharedlineappear=callingsearchspace.pkid where callingsearchspace.name='Long-Distance_CSS'
```

This returned 34 results which matched up with the dependency records.  
3) Determine the pkid of the line CSS that SHOULD be used. The name is "line_long-distance_css"

```
admin:run sql select pkid from callingsearchspace where name="line-long_distance-css"
pkid
====================================
6d5ae38b-bcb3-1453-a3b0-ce328d8c3699
```

4) update these to be the proper pkid. We know from above that the erroneous PKID is `77f5f4de-f98c-1a13-728c-402623cb9ca0` and that the correct PKID is `6d5ae38b-bcb3-1453-a3b0-ce328d8c3699`

```
admin:run sql update numplan set fkcallingsearchspace_sharedlineappear='6d5ae38b-bcb3-1453-a3b0-ce328d8c3699' where fkcallingsearchspace_sharedlineappear='77f5f4de-f98c-1a13-728c-402623cb9ca0'
Rows: 34
```

As you can see, our update statement affected 34 rows, which matches the number of results from step 2. Now we verify that the erroneous CSS isn’t in use.

```
0 Record(s) are using Calling Search Space: Long-Distance_CSS 
```

Looks good! This was a short example here, but SQL saved me a ton of time from manually changing 34 records. I probably could’ve exported the numbers, filtered the excel, and then import/update, but using SQL was MUCH faster.