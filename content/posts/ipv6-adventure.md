---
title: 'Ipv6 Adventure'
author: driz
date: '2025-02-27T18:45:21Z'
tags: ["ipv6"]
categories: ["docker","virt","ipv6"]

summary: Making a bunch of IPv6 related changes brought on by updated in Docker v28.
---

---
title: 'Ipv6 Adventure'
author: driz
date: '2025-02-27T18:45:21Z'
tags: ["ipv6"]
categories: ["docker","virt","ipv6"]

summary: Making a bunch of IPv6 related changes brought on by updated in Docker v28.
---

Today I'm going to discuss a lot of things about IPv6. You may have read my previous article about [IPv6 with docker containers](/posts/ipv6-with-docker-containers/) and I will be changing some things I did there due to new features in [docker v28](https://docs.docker.com/engine/release-notes/28/). First, I'll discuss some why, then a high level of what I've done and then later go into how I did it.

As you may recall in the previous article, I took my single /64 assigned to my trusted vlan and carved it into a couple /112 networks. I then assigned one /112 to docker and one to wireguard. This works fine, however, because my LAN systems only know of the /64 being issued by RA, I couldn't use ipv6 to reach the /112s. Externally this worked because incoming traffic would hit the router and see a static route sending them to the proper downstream devices. I could do this locally on PCs as well by overriding my routes. With v28 of docker though, we gained a lot of cool IPv6 features, specifically routed mode and --ipv4=false. The ipv4=false matters because ipvlan and macvlan networks get interface priority over bridges. I'll expand on this a moment.. when assigning interfaces inside a container, there is an order. It's alphabetical by network name, but ipvlan/macvlan are first, then bridge, then internal. Why does this matter? The default gateway will ALWAYS be the eth0 interface if it exists. So in my case, I wanted ipv4 traffic to flow through a bridge to reach SWAG and other things directly, but I wanted IPv6 to flow through the ipv6 network. in v27, you MUST have an ipv4 network attached, even if you don't use it, so my ipv6 ipvlan network, being the only ipvlan took eth0, this made the default gateway of the container the ipv6 network. Unfortunately, that vlan outside of the container is ipv6 only, so the ipv4 traffic all died but it also prevented bridge to bridge connectivity. With v28, we can set ipv4=false meaning the default gateway created is ONLY for ipv6. The lack of ipv4 on that network also made it not become eth0, though I do not understand why in this case. This allowed inter-container traffic to leverage bridge, outgoing ipv4 traffic to traverse an ipv4 capable network, and forced ipv6 traffic to follow the ipv6 vlan. 

Why did I create a whole vlan for docker ipv6 on my edge? I did this because my ISP then issued an entire /64 to that vlan. I also created a vlan for wireguard. As you can guess, those /112s carved from the trusted network are gone and I'm using dedicated /64s. This also means I didnt need any weird routes for my LAN to reach those networks via ipv6. The IPv6 RFC generally recommends that the smallest network issued out is a /64, so doing this also keeps me in line with those recommendations (though you'll see me break it a bit later). This also allows me to strictly control the firewall for the whole network rather than assuming that things in docker were trusted to be on my trusted lan once they exit the container. 

To get into the high level, some of which is a repeat, I did the following.
1) create ipv6-only vlans for docker and wireguard
2) create firewall rules for those networks to restrict or allow traffic as I see fit
3) upgrade docker to v28 to gain the new ipv6 features
4) swap the ipv6 network for docker to ipvlan
5) update my wireguard configs
6) create some policy based routes on the wireguard server

Let's get into how I did it now. I started on opnsense, which is what I use as my edge router. I created 2 vlans named docker and wireguard, I configured these to be ipv6 only. For docker, I have no intention of ever adding ipv4, but for wireguard, now that I've thorougly tested things, I will eventually add ipv4 and put wireguard on a dedicated system rather than sharing compute as it is now. It actually introduced quite a bit of complexity that I'll discuss later.
![opnsense interface for ipv6](</images/ipv6-adventure/opnsense-interface-ipv6.jpg>)
Once this was complete, I assigned the interface to bring it online and confirmed I was issued two new /64s. At this point, I have /64s and nothing to do with them, the rest of my network isn't aware of these new networks. I trunked the new vlans through to my servers, I dont think I'll need to mention this, but just in case, docker is vlan 300 and wireguard is vlan 301. Once these were trunked through, I realized my docker server has always just been in the trusted vlan on an access port. I had to change this to a trunk port but also set the server up to support vlans. To accomplish this I installed a package called vlan. My primary interface is bond0 (2 10G bonded links) so I needed to make a few adjustments, thank god for IPMI because I lost access numerous times. The original interface looked like this
```bash
auto bond0
iface bond0 inet static
        address 192.168.128.9/24
        gateway 192.168.128.1
        slaves eno1 enp0s25
        bond-mode 802.3ad
        bond-miimon 100
        bond-downdelay 200
        bond-updelay 400
        bond-lacp-rate 1
        mtu 9000
        bond-xmit-hash-policy layer2+3
iface bond0 inet6 auto
```
I changed it to look like this
```bash
#auto bond0
iface bond0 inet manual
        slaves eno1 enp0s25
        bond-mode 802.3ad
        bond-miimon 100
        bond-downdelay 200
        bond-updelay 400
        bond-lacp-rate 1
        mtu 9000
        bond-xmit-hash-policy layer2+3

auto bond0.128
iface bond0.128 inet static
        address 192.168.128.9/24
        gateway 192.168.128.1
iface bond0.128 inet6 auto
auto bond0.300
iface bond0.300 inet6 auto
```
As you can see, I've moved the IP configuration to the sub-interfaces accordingly. Vlan 128 is the trusted vlan, vlan300 as you may recall is docker. I leave the bonding configuration back on the primary interface. After some hiccups and some reboots, this was working
```bash
docker proxy-confs # ip a show dev bond0
4: bond0: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 9000 qdisc noqueue state UP group default qlen 1000
    link/ether 00:25:90:a2:d1:48 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::225:90ff:fea2:d148/64 scope link
       valid_lft forever preferred_lft forever
docker proxy-confs # ip a show dev bond0.128
5: bond0.128@bond0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc noqueue state UP group default qlen 1000
    link/ether 00:25:90:a2:d1:48 brd ff:ff:ff:ff:ff:ff
    inet 192.168.128.9/24 brd 192.168.128.255 scope global bond0.128
       valid_lft forever preferred_lft forever
    inet6 2600:1702:1b20:3c40:225:90ff:fea2:d148/64 scope global dynamic mngtmpaddr
       valid_lft 86386sec preferred_lft 14386sec
    inet6 fe80::225:90ff:fea2:d148/64 scope link
       valid_lft forever preferred_lft forever
docker proxy-confs # ip a show dev bond0.300
6: bond0.300@bond0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc noqueue state UP group default qlen 1000
    link/ether 00:25:90:a2:d1:48 brd ff:ff:ff:ff:ff:ff
    inet6 2600:1702:1b20:3c42:225:90ff:fea2:d148/64 scope global dynamic mngtmpaddr
       valid_lft 86234sec preferred_lft 14234sec
    inet6 fe80::225:90ff:fea2:d148/64 scope link
       valid_lft forever preferred_lft forever
```
This was my first time using ipvlan, because prior to v28, it can do some weird shit to your networks and in my opinion a custom bridge is almost always the superior choice. Here is how I created the network:
`docker network create -d ipvlan --ipv4=false --ipv6 --subnet 2600:1702:1b20:3c42::/64 --gateway 2600:1702:1b20:3c42::1 -o parent=bond0.300 -o com.docker.network.bridge.gateway_mode_ipv6=routed ipv6`
This gives me an IPv6 ONLY network that routes rather than the typical docker method of natting. I'll show snippets from my compose and then some other docker outputs
```bash
  smokeping:
    <<: *lsioint
    image: lscr.io/linuxserver/smokeping:latest
    container_name: smokeping
    networks:
      dznet:
      ipv6:
        ipv6_address: 2600:1702:1b20:3c42::3
    volumes:
      - ${CONFDIR}/smokeping/config:/config
      - ${CONFDIR}/smokeping/data:/data
networks:
  dznet:
    external: true
  ipv6:
    external: true
```
Note that I have 2 routable IPv6 addresses because one is issued by RA and one is the statically assigned IP from compose.
```bash
docker exec -it smokeping ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0@if182: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 06:1b:15:2b:dc:6a brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.5/16 brd 172.18.255.255 scope global eth0
       valid_lft forever preferred_lft forever
206: eth1@if6: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 9000 qdisc noqueue state UNKNOWN
    link/ether 00:25:90:a2:d1:48 brd ff:ff:ff:ff:ff:ff
    inet6 2600:1702:1b20:3c42:25:9000:8a2:d148/64 scope global dynamic flags 100
       valid_lft 86344sec preferred_lft 14344sec
    inet6 2600:1702:1b20:3c42::3/64 scope global flags 02
       valid_lft forever preferred_lft forever
    inet6 fe80::25:9000:8a2:d148/64 scope link
       valid_lft forever preferred_lft forever
```
At this point, I did some outbound and inbound tests which I won't paste here, but I tested inbound pings directly to the container ip, they work. I tested ipv6 site readiness, passed. I tested that I could access, via ipv6, external resources from within the container, it worked. I repeated my smokeping steps for Plex and Swag, updated my AAAA records on cloudflare and the docker portion is completed. I have no static routes on my router or any hosts, it all just works.

Wireguard was a different mess, and honestly it is likely due to me not knowing the best ways to handle the situation. Where docker has ipvlan that lets the container appear like just another system on the network, I couldn't do this with my bare-metal wireguard instance. Some things were simpler some were more difficult. In this case, my wireguard node is a vm, so i simply added a new vNic assigned to vlan300. I did not install the vlan package on the node, which likely caused some of the issues with routing. Basically, the issues were the default gateways... A packet could come from a wireguard client, but rather than following the wireguard vlan, it would follow the trusted vlan and this worked, traffic reaches the destination, however, the return traffic (acks) would see the destination was a wireguard client and follow the proper directly connected route for wireguard causing asymmetric routing which would cause the communications to fail. We will get into that soon.
Moving on, I've added my vNic which linux immediately picked up, so now I need to configure it.
```bash
iface ens192 inet static
 address 192.168.128.10
 netmask 255.255.255.0
 network 192.168.128.0
 gateway 192.168.128.1
 post-up ip link set mtu 9000 dev ens192
iface ens192 inet6 auto

auto ens224
iface ens224 inet6 auto
```
I did this, and I had an ipv6 address on ens224. I updated my wg0.conf and client configs to use the new subnet and things... didn't work from an ipv6 perspective. Part of the issue is that the IPs I assign to clients aren't issues by RA so the router has no knowledge of them, meaning, return traffic couldn't reach them. In this case, I took a /124 from the /64 for wireguard and created a static route on opnsense for this. In this case, since the wireguard ipv6 subnet needed to be routed to anyway, even internal clients saw the static route on the router during inter-vlan communications. This also allowed the router to be aware of this subnet i chunked out. I'm sure there is some way to handle this, but unfortunately, I am not sure what that method is. Anyway, I removed ipv4 from the client configs and forced ipv6 only and immediately hit issues. The asymmetric routing was the first, but the wireguard server is also home to the unbound and pihole servers. I was seeing pihole receiving communications on ens192 and then sending them to unbound on ens224 when it should've been using ::1 or 127.0.0.1. I fixed this in unbound with the `interface` and `access-control` options, as only pihole should be speaking to unbound. After this, I realized the root cause of the issues was the default route. I decided to try some policy based routing to work around this issue. For whatever reason, the default routes assigned by RA were prioritizing ens224, even if I manually fixed this, then traffic that should be on ens192 was egressing via ens224 because.. well that is how default routes work. So I decided to make policies that say if the source is this network, egress this interface then left a normal default route for the rest, but with a higher metric. Below you'll see my commands, note that I did not delete any of the routes or default routes created by RA because their metrics are over 1000, so I just created new routes with much lower metrics (better preference)
```shell
ip -6 rule add from 2600:1702:1b20:3c43::/64 lookup wgnet
ip -6 route add default via fe80::227c:14ff:fef4:a7bc dev ens224 metric 100 table wgnet
ip -6 route add default via fe80::227c:14ff:fef4:a7bc dev ens192 metric 110
```
The first command creates a table that says the source is the network defined and it's labeled wgnet (in iproute2/rt_tables), next we create a default route to ens2 with a very low metric, restricted to the table we created. So any traffic from that source network will always prefer the default route created. Next, with a slightly higher metric, I made a default route for everything else. Using tcpdump (I'll share the command below) I confirmed the traffic was routing as I intended.
`tcpdump -i any ip6 and port 443 and net 2600:1702:1b20:3c42::/64 and net 2600:1702:1b20:3c43::/64` I used this one to check wireguard to docker (i was connecting from a wireguard client to something behind swag and the ipv6 is direct, no natting or other interfaces)
`tcpdump -i any ip6 and port 443 and net 2600:1702:1b20:3c42::/64 and net 2600:1702:1b20:3c40::/64` then I used this one to confirm the box itself used the proper interfaces when reaching swag.  Everything was good and fully functional, I rebooted for good measure. did you guess it? yah, box came back up, nothing worked. tcpdump shows everything is wrong again.. my routes were all gone, so I realized I needed to persist them, which in retrospect is obvious.
```shell
allow-hotplug ens192
iface ens192 inet static
 address 192.168.128.10
 netmask 255.255.255.0
 network 192.168.128.0
 gateway 192.168.128.1
 post-up ip link set mtu 9000 dev ens192
iface ens192 inet6 auto
  post-up ip -6 route add default via fe80::227c:14ff:fef4:a7bc dev ens192 metric 110

auto ens224
#allow-hotplus ens224
#iface ens224 inet static
# post-up ip link set mtu 9000 dev ens224
iface ens224 inet6 auto
  post-up ip -6 rule add from 2600:1702:1b20:3c43::/64 lookup wgnet
  post-up ip -6 route add default via fe80::227c:14ff:fef4:a7bc dev ens224 metric 100 table wgnet
```
I rebooted again after making the above change and everything worked. Note that when you want to view your routes, by default `ip -6 route show` will only show the local table, which is default. If you want to see your wgnet table, you must specify `ip -6 route show table wgnet`.

At this point, everything is working using native IPv6. Things that should be directly accessible with no nat are directly accessible with no nat, and I'm relatively happy with what I've learned and implemented! I hope this is useful information!