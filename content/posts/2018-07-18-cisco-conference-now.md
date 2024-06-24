---
title: Cisco Conference Now
author: driz
type: posts
date: 2018-07-18T20:35:00+00:00
slug: cisco-conference-now
aliases: /2018/07/cisco-conference-now/
categories:
  - Cisco Unified Communications
  - voice
summary: |
  With Call Manager 11x we saw the deprecation of Meet Me conferences begin. Meetme conferences were great, but many users had issues with using them. This is likely what led to the mass exodus of users to things like webex, zoom, and bluejeans. Today was the first time I’ve ever had the opportunity to work on Conference Now, so I will run through what I did to get this all working.

  The photos I will use in the guide are from UCM 12.5, but the process is the same in 11x.

  ![](https://www.cisco.com/c/dam/en/us/support/docs/unified-communications/unified-communications-manager-callmanager/200181-Configure-Conference-Now-Feature-on-CUCM-00.png))
---
With Call Manager 11x we saw the deprecation of Meet Me conferences begin. Meetme conferences were great, but many users had issues with using them. This is likely what led to the mass exodus of users to things like webex, zoom, and bluejeans. Today was the first time I’ve ever had the opportunity to work on Conference Now, so I will run through what I did to get this all working.

The photos I will use in the guide are from UCM 12.5, but the process is the same in 11x.

![](https://www.cisco.com/c/dam/en/us/support/docs/unified-communications/unified-communications-manager-callmanager/200181-Configure-Conference-Now-Feature-on-CUCM-00.png))

First, we need to setup the base functionality by going to **Call Routing -> Conference Now**

Fill in your information; if you use a dialplan with less than 10 digits (shame on you) simply put the appropriate extension in the DN box.

![](/images/cisco-conference-now/image-3.png))

Next, go to **Media Resources -> Interactive Voice Response**, verify your IVR is registered to the UCM. Verify your Device pool, location, and descriptions are set properly.

Since IVRs are a media resource, you can add them to an MRG and subsequently an MRGL. Under **System -> Service Parameters**, select a voice processing node and Cisco IP Voice Media Streaming App to confiure your call count and run flag. By default, these will be 48 and true, which are perfectly fine to start.

![](/images/cisco-conference-now/image-4.png))

You can verify your announcements are present by going to **Media Resources -> Announcements**. You should see the basic ConferenceNow announcements. You can change these by clicking the announcement and uploading a new wave file.

Next, We go to **User Management -> User/Phone Add -> Feature Group Template**. Let’s enable end user to host conference now and save.

For the end user specifically, there are a few things here that matter. We want to ensure the end user is enabled to host, that they know their meeting number (it’s the same as their extension and self-service user-id by default) and that they have an access code if they want it (users can set their access code by visiting the self-service portal)

Functionally, the way this works, someone dials in to the DID (or internal number). When they dial-in they are prompted for the meeting number, remember, by default the meeting number is the same as the users extension. If they are not the host (read: does not know the PIN set under the end-user in UCM) they will listen to MOH until the host joins, or be disconnected after 15 minutes ( based on the max wait time we set above). When the host joins, they simply input their pin and the conference begins.

The issue I ran into was enabling ALL end users to host conference now on an existing system. With over 1000 users, you immediately might think of BAT. Well, you can’t BAT enable end users to host conference now. The next option would be to export end users and change the f to t and reimport. However, this will fail with an error unless you also modify the PIN AND the digest credentials (for some reason) which could wreak havoc on a system where extension mobility is in use.

My solution, as you might have guessed was to use SQL. To start, I ran a query to ensure it would work

![](/images/cisco-conference-now/image-5.png)

As we can see, my 4 users and some strange token user showed up. We can see enable user to host conference now is set to false. My first 3 attempts to update failed; i tried to knock it out in one fell swoop without much luck

![](/images/cisco-conference-now/image-6.png)

So, we see that we can’t simply affect all users, use a wildcard user, or change it to t if it’s currently f (which I find to be stupid). So I tried changing just one user, which worked.

![](/images/cisco-conference-now/image-7.png)

My next great idea was ok, this worked, let’s change every user that ISN’T wlongo

![](/images/cisco-conference-now/image-8.png)

This worked and set all my users how I wanted. A more responsible way to do this would probably be to add a second where clause to specify `enableusertohostconferencenow='f'`, but I didn’t do that.

Let’s verify our work

![](/images/cisco-conference-now/image-9.png)

Everything looked good, so I went ahead and did the same thing on the client system with 1000 users. It took about 5 seconds to complete. I spot tested a couple of the test users by dialing in and specifying their meeting code. This works because if they aren’t enabled to host, the meeting number will fail. If it works, you know they’re setup properly 🙂

Anyway, hopefully this helps someone