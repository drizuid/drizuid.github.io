---
title: CRASH! Getting root on Cisco UC Devices without TAC!
author: Will
type: posts
date: 2019-09-26T15:19:00+00:00
slug: root-access-on-cisco-uc-devices
aliases: /2019/09/root-access-on-cisco-uc-devices/
categories:
  - Cisco Unified Communications
  - Linux
  - Operating Systems
  - voice
  - womp
summary: In this article, I discuss how to get root access to Cisco UC Applications, without leveraging TAC. This is NOT a supported process.
---
NOTE: Due to poor backups, I lost this post. I was able to restore it thanks to the Internet Archive, but I lost all images. I will try to restore these at a later time.  
  
I was working with a client and we were doing a parallel build in their environment. To do this, i build the new environment without vnics connected, then i disconnected production and connected the new. However, we ran into some issues and had to revert. While working with tac on the DISCONNECTED nodes, they wanted root. They ran their usual remote_account command to get the code, however, you can't copy paste from a vmware console. To work around this issue, i booted from a centos 8 minimal (around 1.6GB) disc.  
  
I selected **troubleshooting -> rescue mode**. The disc loaded into memory and prompted to ask if it should continue with mounting the system to be rescued to /mnt/sysimage. I pressed `1` and hit enter to do so. Once that finished, i typed `chroot /mnt/sysimage\` this put me into a chroot jail on the actual OS of the system (i'm inside the call manager)  
  
From here I did a few things.  
1) set a password on the root account with `passwd root`
2) change the shell from `/sbin/nologin` to `/bin/bash` in `/etc/passwd` for root  
3) change `permitrootlogin` in `/etc/ssh/sshd_config` to yes.  
  
Once I completed these changed, I exited my chroot environment, unmounted my cd, and rebooted the system. Once the system booted up, i was able to successfully login as root and allow TAC to continue their work.