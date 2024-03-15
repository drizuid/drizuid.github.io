---
title: Routing containers through Wireguard
author: Will
type: posts
date: 2024-03-14T17:51:15.000Z
slug: routing-containers-through-wireguard
aliases: /2024/03/routing-containers-through-wireguard/
tags: ["docker"]
categories: ["docker","virt"]
draft: false
summary: In a regular day on the linuxserver.io discord, we have a lot of people come in with weird vpn setups or just terrible network configurations. They inevitably want to know how to route their torrent client of choice through a vpn while still being able to access the web ui and have their other tools access the client, without also going through the VPN. I've always considered this to be relatively simple basic networking and have never given it additional thought. However, with the prompting of some friends/colleagues, I decided to give it a go and see how things went. 
---

In a regular day on the linuxserver.io discord, we have a lot of people come in with weird vpn setups or just terrible network configurations. They inevitably want to know how to route their torrent client of choice through a vpn while still being able to access the web ui and have their other tools access the client, without also going through the VPN. I've always considered this to be relatively simple basic networking and have never given it additional thought. However, with the prompting of some friends/colleagues, I decided to give it a go and see how things went.  

As an affiliate for torguard, I decided I would use them for my efforts. Some things to note is that they allow port forwarding on the vpn, as long as you select a non-US server, they have stellar discounts, and they've proven themselves to be quite reliable while offering competitive speed. I would be remiss if I didn't shamelessly drop my [affiliate link here](https://torguard.net/aff.php?aff=3427). You can also use voucher `LSIO` to save 50% on your purchase, for life.

Without further ado, I'll get into the meat of the article. Initially, I thought that I would also be using IPv6, so I proceeded to set this up. Using my typical methodology, I carved out a /112 from my block and created the custom bridge `docker network create --subnet 172.20.0.0/24 --ipv6 --subnet 2600:1700:6cb4:6012:3::/112 wgnet` Unfortunately, I later realized that torguard doesn't support ipv6 on their wg vpn yet. Considering this further, I probably shouldn't have used public ipv6 space anyway, so if they ever offer support, I will use something in the ULA range instead, since the routable IP is what the provider will give us anyway. 

Next I created a directory to store my wireguard stuff. Throughout this article, I'll reference `${CONFDIR}` so let's just define it now, `${CONFDIR}` is simply `/srv/docker`. `mkdir ${CONFDIR}/wg-client`. 

I then signed up for the 3 year wireguard plan, using my 50% voucher it came out to something like $67.99 every 3 years. Unfortunately, right after I signed up, I realized I had a 60% voucher code which expires on March 15th, so I lost out on some savings, oops. Once I finished this, I signed into TG and setup my port forwarding **Services -> manage -> port forward request**

The interface is a bit confusing, but the key points are forward your torrent port, in my case, 51413. I did both udp and tcp. On the right side, input your public ip that wireguard provided you and yes, port forwards are per ip, not account. Port/auth will be wireguard and as you may know, wireguard's protocol is UDP.
![port forward request](/images/routing-containers-through-wireguard/port_forward.png)

If you wait a few minutes, you should receive an email confirming your ports are forwarded. It took about 2 minutes for me to get mine. You can also confirm by checking your request status, it should say active.
![request approved](/images/routing-containers-through-wireguard/approved.png)

Once that is complete, use the (config generator)[https://torguard.net/tgconf.php?action=vpn-openvpnconfig], you'll select wireguard for tunnel type, select your port forward ip from the list (fixed ips), input your vpn username (it's your TG username/email and is also shown in your welcome email). Other than that, I only changed dns to the vpn provider's dns. Generate your config and paste the contents into wg0.conf in your previously created directory (`${CONFDIR}/wg-client in my case).
![config generator](/images/routing-containers-through-wireguard/config-gen.png)

I created my wireguard container and assigned it to the new network. 
```yaml
  wireguard:
    <<: *base
    image: lscr.io/linuxserver/wireguard
    container_name: wireguard
    healthcheck:
      test: ['CMD-SHELL', "[[ $(curl -s https://icanhazip.com/) != $(nslookup ${DOMAIN} | awk -F': ' 'NR==6 { print $2 } ') ]]"]
      start_period: 60s
      interval: 30s
      retries: 5
      timeout: 10s
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      <<: *lsio
    volumes:
      - ${CONFDIR}/wg-client:/config
      - /lib/modules:/lib/modules
    networks:
      wgnet:
        ipv4_address: 172.20.0.50
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    #ports:
    #  - 9091:9091
    #  - 51413:51413
    #  - 51413:51413/udp
```
In my case, I statically assigned IPs for a little extra control. My yaml anchors are just a diun label, a memory limit, a restart policy, and further down, PUID/PGID, and TZ. I already know only transmission is going through this container, so I've prepped my ports accordingly, but commented them out to avoid a port conflict. The healthcheck will compare my public ip as indicated by my DDNS A record to what icanhazip reports my ip as. If they match, something bad is going on, if they dont match, the vpn connection is established.

I brought up my container and saw no errors. At this point, I did a small test `docker exec -it wireguard curl -s icanhazip.com` and i got back my vpn ip. Just in case, I checked with -4 and -6 specified, nothing for ipv6, as expected. With preliminary tests looking good, it's time to move into the next step, putting transmission through wireguard.

To accomplish this, I first uncommented the ports on wireguard above, and modified transmission to remove ports and networks, but add my vpn service.
```yaml
  transmission:
    <<: *base
    image: lscr.io/linuxserver/transmission
    container_name: transmission
    healthcheck:
      test: curl --fail curl --fail http://${TRANSUSER}:${TRANSPASS}@localhost:9091 || exit 1
      interval: 20s
      retries: 5
      start_period: 20s
      timeout: 10s
    network_mode: service:wireguard
    depends_on:
      wireguard:
        condition: "service_healthy"
    environment:
      <<: *lsio
      USER: ${TRANSUSER}
      PASS: ${TRANSPASS}
      S6_KILL_GRACETIME: 30000
    volumes:
      - ${BDDIR}/docker/transmission:/config
      - ${BDDIR}/BT:/downloads
      - ${TIME}:${TIME}:ro
```

The network_mode above removes network adapters and makes it rely on the wireguard container for service. If the wireguard container is down, the "nic" is disconnected. With the healthcheck above and our depends_on in transmission, transmission will only start if the vpn is active. There are likely some issues where maybe the vpn disconnects and I leak in between the healthcheck, but I'll worry about that later.

I run an up -d on wireguard and transmission with my changes. The web ui became inaccessible as did the rpc client. This is because 9091 is no longer available on the network. I do a test in transmission `docker exec -it transmission curl -s icanhazip.com` and i got back my vpn ip. So, it's working as intended.

Now I need to make some adjustments to the network config of wireguard to ensure I can connect with my client on the lan. I stole this, with slight modifications, from my teammate Quietsy's [blog]( <)https://virtualize.link/vpn/) in my wg0, above the peer section, I add the following
```Shell
PostUp = DROUTE=$(ip route | grep default | awk '{print $3}'); HOMENET=192.168.128.0/24; HOMENET2=172.16.0.0/12; ip route add $HOMENET2 via $DROUTE; ip route add $HOMENET via $DROUTE;iptables -I OUTPUT -d $HOMENET -j ACCEPT;iptables -A OUTPUT -d $HOMENET2 -j ACCEPT; iptables -A OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT
PreDown = DROUTE=$(ip route | grep default | awk '{print $3}'); HOMENET=192.168.128.0/24; HOMENET2=172.16.0.0/12; ip route del $HOMENET2 via $DROUTE; ip route del $HOMENET via $DROUTE; iptables -D OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT; iptables -D OUTPUT -d $HOMENET -j ACCEPT; iptables -D OUTPUT -d $HOMENET2 -j ACCEPT
```
`192.168.128.0/24` is my LAN network that I would be connecting from (later I may add my personal vpn network so if i'm remote I can see). I also put in the whole 172 range to cover other containers connecting in, just in case. What this does is create routes for these specific networks to the default route. It created iptables rules in the output chain to allow connectivity to those networks, and then rejects packets in the oytput chain that have the fwmark identifier in wg from going over any interface aside from wg0. Of course we need the inverse too, so when wg is going down, we remove the rules.

With those rules added, I restart wireguard (I tested transmission during this time to see if it had wan access, it did not, which is precisely what we want). I checked my container logs and see the iptables and route rules take effect. Now I can access the webui and use rpc clients on transmission. The reason for this is that the subnets specified in the above rules bypass the default route over wg0, so my desktop goes to the docker host which listens on port 9091, for example, nats it to wireguard (now because we stopped mapping that port on transmission), which then sends it to transmission. Without the rules above, the reply would go out wg0 and just die after ttl expiration and we would see nothing.

There are a few items to resolve now, swag, cross-seed, and the arr suite. to resolve these, I simply add `wgnet` to the networks of those containers in my compose. that will allow them to use that network to communicate with other things on that network. I do ann up -d on those to ensure the new networks are present. Here is a shortened example of the change
```yaml
services:
  radarr:
    <data removed for ease of reading>
    networks:
        servarr:
        wgnet:  #<--this is the new addition
```

For SWAG, I change the upstream_app variable from "transmission" to "wireguard" because for swag to reach transmission, it must go to http://wireguard:9091 now rather than http://transmission:9091, as the port is only exposed via wireguard. Here is a snippet of that change
```Shell
        set $upstream_app wireguard; #previously this was transmission
        set $upstream_port 9091;
        set $upstream_proto http;
```
For cross-seed, I needed to modify my config.js to connect to wireguard rather than transmission which looked like this originally `transmissionRpcUrl: "http://user:password@transmission:9091/transmission/rpc",` and now looks like `transmissionRpcUrl: "http://user:password@wireguard:9091/transmission/rpc",`. I'm sure you can already surmise what was needed for the arr suite. change the download client from host: transmission to host: wireguard. HOWEVER, in my case, i needed an extra step. I do not use consistent folder naming amongst my containers, I rely on remote path mappings. This basically lets you say that the transmission path is /downloads/Movies but the ARR path is /media/BT/Movies. In this setup though, you specify the host that this mapping applies to (basically which download client is this for). Since i changed my download client from transmission to wireguard, my remote path mapping is invalid because there is no host transmission. I clicked the drop down and selected wireguard on each app. 
![download client in radarr](/images/routing-containers-through-wireguard/edit-client-radarr.png)
![Path Mapping in radarr](/images/routing-containers-through-wireguard/path-mapping-radarr.png)

Now everything is working. If wireguard is down, transmission has no network access, if wireguard is up, the ip is confirmed before allowing transmission access. 