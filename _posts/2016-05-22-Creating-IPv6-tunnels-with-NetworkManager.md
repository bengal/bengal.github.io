---
layout: post
title: Creating IPv6 tunnels with NetworkManager
tags: [ networkmanager, linux, ipv6 ] 
---

According to [these statistics] [1], in April 2016 approximately only
10% of users reached Google servers through IPv6. This shows that IPv6
adoption is steadily increasing, but still quite low despite the good
support available in the network infrastructure and all operating
systems.

For users who would like to try IPv6 but are stuck with IPv4 due to
lack of support from their ISPs, there are free **tunnel broker**
services, which, as explained in [RFC 3053] [2], are *"virtual IPv6 ISPs,
providing IPv6 connectivity to users already connected to the IPv4
Internet"*. *"Tunnel"* refers to the fact that IPv6 packets from the
host are encapsulated in IPv4 packets to reach the broker through the
internet (and vice-versa). One of the most used tunnel brokers is
[Hurricane Electric] [3], which is also the one I've used in this
post; but the setup should be quite the same for every other provider.

NetworkManager 1.2 supports the creation of a broad range of software
devices including VXLANs, MAC-VLANS/MAC-VTAPs, TUN/TAP and, of course,
IP tunnels. After registering on the provider website and creating a
new tunnel instance, the NetworkManager command line utility (*nmcli*)
can be used to create a new tunnel in the following way:

<!--more-->

    $ nmcli connection add \
            type ip-tunnel ifname he-sit \
            mode sit remote 216.66.86.114 -- \
            ipv4.method disabled ipv6.method manual \
            ipv6.address 2001:470:6c:dd3::2/64 \
            ipv6.gateway 2001:470:6c:dd3::1

where:

 - "`ifname he-sit`" defines an arbitrary name to assign to the tunnel interface
 
 - "`mode sit`" specifies the tunneling protocol (one among sit, ipip, gre, ipip6, ip6ip6)
 
 - "`remote 216.66.86.114`" is the IPv4 endpoint of the tunnel as
   indicated by the provider

 - "`ipv6.address 2001:470:6c:dd3::2/64`" sets the IPv6 address to be
   assigned to the local interface, as indicated by the provider

 - "`ipv6.gateway 2001:470:6c:dd3::1`" sets the IPv6 address of the
   gateway, as indicated by the provider

The example above only sets the most common parameters, however
advanced users can add more options as documented in the
`nm-settings(5)` man page.

Users with a dynamic IPv4 address must notify the broker of their
public address, and this can be done manually from the web interface
or automatically through a cron job. But the simplest way do it is
through a NetworkManager dispatcher script that will update the
address every time the tunnel is established.

For a Hurricane Electric tunnel, the following dispatcher script can
be put in `/etc/NetworkManager/dispatcher.d/pre-up.d/` and will be
called just before declaring the tunnel as established (remember to
set it as executable using "`chmod +x <script>`"):

    #!/bin/sh

    USER=myuser
    PASSWD=mypasswd
    TUNNEL_ID=123456

    if [ "$1" = he-sit ]; then
        wget -O /dev/null https://$USER:$PASSWD@ipv4.tunnelbroker.net/ipv4_end.php?tid=$TUNNEL_ID
    fi

where *USER*, *PASSWD* and *TUNNEL_ID* are parameters available on the
tunnel configuration page. Remember to configure firewall rules on the
interface because your machine will be reachable from Internet without
any filtering.

[1]: http://www.google.com/intl/en/ipv6/statistics.html
[2]: https://tools.ietf.org/html/rfc3053
[3]: https://www.tunnelbroker.net/
