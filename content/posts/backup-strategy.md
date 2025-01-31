---
title: 'Backup Strategy'
author: driz
date: '2025-01-31T14:13:01Z'
tags: ["backups"]
categories: ["docker","backup"]

summary: I wanted to share how I have historically been handling backups. I have plans to try out things like restic at some point, but since everything is working, I haven't put the time in to do so. I currently rely on bash scripts which leverage rclone to backup to gDrive, via cron. 
---

I have a small number of servers, all with things that need to be backed up to ensure I don't lose data or functionality. As you may recall from the [database loss I suffered in 2020](/posts/a-sad-day/), backups are critical, but TESTING backups is equally important! This won't be a very long post, but I will show my methodology.

For mail on my systems, I leverage exim4 and since I've forgotten how to use exim4 10+ years ago, the backups are quite important to avoid starting over. To backup these configs, below is my script, which I will break down after.

```bash{linenos=true}
#!/bin/bash

DIRECTORY="/etc/exim4"
FILE="$DIRECTORY/bak/exim4-$(date '+%Y%m%d-%H%M')".txz
TMP="/srv/dev-disk-by-label-Big-Data/tmp"
PASS=/root/.env
. "$PASS"

curl -fsS --retry 3 https://healthchecks.$domain/ping/<unique code>/start
#
find /etc/exim4/bak/ -maxdepth 1 -type f -printf '%Ts\t' -print0 | sort -rnz | tail -n +4 -z | cut -f2- -z | xargs -0 -r rm -f

#compare hash to determine if there was a change
cd $DIRECTORY/
find . -type f ! -path "./bak/*" \( -exec md5sum "$PWD"/{} \; \) | awk '{print $1}' | sort | md5sum > $TMP/exim4currsum
if ! cmp -s $TMP/exim4currsum $TMP/exim4oldsum
then
        tar cf - --exclude="bak" $DIRECTORY /etc/email-addresses /etc/aliases | xz -z -9ve -T0 - > $FILE
        gpg --yes --batch --passphrase=$OTHERPW -c $FILE
        rm $FILE
        /usr/bin/rclone sync /etc/exim4/bak GDrive:backups/dznet-nas/etc/exim4
fi
rm $TMP/exim4oldsum
mv $TMP/exim4currsum $TMP/exim4oldsum

#
curl -fsS -m 10 --retry 5 -o /dev/null https://healthchecks.$domain/ping/<unique code>/$?
```

1. looking at the code, I set a few variables, the directory we are concerned with, the filename I will use for the backup file, a tmp location, and a password.
```bash{linenos=true}
DIRECTORY="/etc/exim4"
FILE="$DIRECTORY/bak/exim4-$(date '+%Y%m%d-%H%M')".txz
TMP="/srv/dev-disk-by-label-Big-Data/tmp"
PASS=/root/.env
. "$PASS"
```
2. Next we have a curl to healthchecks, this is how I ensure the cron tasks execute successfully. I get alers via email (weekly) and immediately via discord. This first curl lets healthchecks know that the script is running and the final curl sends the exit state which is a numeric code indicating success or failure.

`curl -fsS --retry 3 https://healthchecks.$domain/ping/<unique code>/start`

3. Prior to conducting the backup, I do a little cleanup, I only maintain the most recent 3 backups for this particular application, which hasn't changed in years. 

`find /etc/exim4/bak/ -maxdepth 1 -type f -printf '%Ts\t' -print0 | sort -rnz | tail -n +4 -z | cut -f2- -z | xargs -0 -r rm -f`

4. Now we get into the backup process, before we create a new backup file, we check for a change. I leverage md5 to do this, sha will also work but takes longer and I am not salting anything important, just comparing to know if a change happened. This particular line checks the directory as a whole, excluding the actual backup directory, and outputs the sum to exim4currsum in the tmp directory.
```bash{linenos=true}
cd $DIRECTORY/
find . -type f ! -path "./bak/*" \( -exec md5sum "$PWD"/{} \; \) | awk '{print $1}' | sort | md5sum > $TMP/exim4currsum
```
5. Now I check, via if, if the current sum we just took is different from the oldsum which stays in the tmp directory. If it is different, we will tar, excluding the backup directory, the defined directory and 2 files from /etc and then run them through xz to output to the filename specified, which includes a date.
```bash{linenos=true}
if ! cmp -s $TMP/exim4currsum $TMP/exim4oldsum
then
        tar cf - --exclude="bak" $DIRECTORY /etc/email-addresses /etc/aliases | xz -z -9ve -T0 - > $FILE
```
6. Since the contents here had sensitive information, such as the password for my e-mail account, I use my private key to encrypt the contents, via GPG. We then remove the original unecrypted file. OTHERPW is simply an item within the .env that is pulled in at the top.
```bash{linenos=true}
        gpg --yes --batch --passphrase=$OTHERPW -c $FILE
        rm $FILE
```
7. Finally, we leverage rclone to sync our local backup directory to google drive.

`        /usr/bin/rclone sync /etc/exim4/bak GDrive:backups/dznet-nas/etc/exim4`

8. If there was no change in the hash sum, the if ends and we do cleanup. We also do this cleanup if the hash was different, once the backup completes, this involves removing the original old hash sum and moving the new one to the exim4oldsum file. 
```bash{linenos=true}
rm $TMP/exim4oldsum
mv $TMP/exim4currsum $TMP/exim4oldsum
```
9. To close things out, we send our exit code from the script to healthchecks, which lets it know the script finished running and if it was successful or failed.

`curl -fsS -m 10 --retry 5 -o /dev/null https://healthchecks.$domain/ping/<unique code>/$?`

Here are the current contents of the bak/ folder referenced above
```bash
 bak # ls
total 56K
drwxr-xr-x 2 root root 4.0K Jan 24 08:00 .
drwxr-xr-x 4 root root 4.0K Dec  5 09:51 ..
-rw-r--r-- 1 root root  23K Jan 23 08:10 exim4-20250123-0810.txz.gpg
-rw-r--r-- 1 root root  23K Jan 24 08:00 exim4-20250124-0800.txz.gpg
```
There hasn't been a new file since January 24th because the hash sum has not changed since then. These two files were actually just created as samples in preparation for writing this article.


You can see the output of healthchecks in discord below
![healthchecks success](</images/backup-strategy/exim4 healthcheck.jpg>)

I do the same strategy for backing up most of my applications, containers, and even personal documents. As you may be aware from other articles, my family's desktops and laptops all have the "documents", "desktop", and "pictures" folders mapped to the NAS. These folders are also backed up to each user's individual Google drive.

Anyway, hopefully this is useful information, though if you are just getting started in your backup adventure, using a mature tool like restic is probably a better idea.