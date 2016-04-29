---
layout: post
title: NetworkManager 1.2 and DNS
---

 NetworkManager 1.2 was [recently released] [1] after more than a year
 of development and is available in major distributions including
 Fedora 24 and Ubuntu 16.04. In this post I will briefly talk about the
 features NetworkManager offers to users for configuring DNS on their
 machines, and how this has improved in the last release.

<!--more-->
 
## What's new

 Let's first have a quick look at the new features added in
 NetworkManager 1.2 and how users can benefit from them.

### Better cooperation with other tools

 A classic problem that arises when managing systems where multiple
 network configuration tools coexist is how to ensure that they don't
 interfere with each other in updating *resolv.conf*.

 NetworkManager by default now installs *resolv.conf* as a symbolic
 link to a file in a private directory, and will refuse to touch the
 file if it's a link pointing to a different location. In this way
 system administrators can decide which tool is allowed to manage the
 file by changing the target of the link.

### DNS options

 The behavior of the C library resolver can be modified by setting
 options in *resolv.conf* to allow users to fine-tune some parameters
 like the timeout for DNS queries, the number of retries, etc. (see
 "man resolv.conf" for a complete list of supported options).

 In the past there was no simple way to change this options since
 NetworkManager would overwrite them in *resolv.conf*. Now it is
 possible to set per-connection DNS options.

### Global DNS configuration

 For environments with DNS parameters that are never supposed to
 change, the only way to set a static configuration before 1.2 was to
 manually update *resolv.conf* with the desired parameters and make
 NetworkManager not manage the file. Now a global DNS configuration
 can be set in *NetworkManager.conf*, with some advantages over the
 old approach:

 * a static configuration with dnsmasq and split DNS can be used

 * the static configuration can be retrieved and updated through D-Bus

NetworkManager reads configuration snippets from its `conf.d`
directory, so it is also possible to place the static configuration in
a separate file which can be pre-installed and deployed to multiple
machines.

### A signal to force reconfiguration

 If you ever wondered how to force NetworkManager to rewrite
 *resolv.conf* after you messed up with it, now the `SIGUSR1` signal
 can be used to achieve that.

### rc-manager

 NetworkManager supported since some time alternative ways of writing
 *resolv.conf*. By default it updates the file directly, but it is
 also possible to do it through external tools like *resolvconf* and
 *netconfig*. However, it was not very practical to switch between
 these as a recompilation with different options was required.

 A new *'rc-manager'* option is now available to select the
 *resolv.conf* update method at runtime; the option is also reloaded
 after `SIGUSR1` and thus a restart of the daemon is not required!

## What's coming next

 This isn't all; in upcoming releases there will be other improvements
 that were requested by users.

 For example, NetworkManager will be able to update dnsmasq
 configuration through D-Bus, which is more efficient than restarting
 the daemon at every configuration change.

 Another feature often requested is a way to tell NetworkManager to
 use only DNS servers from a single connection, ignoring others. This
 is usefully especially for VPNs, where users want to avoid leaking
 DNS queries to name servers different from the one on the VPN. This
 feature will be implemented through a "DNS priority" per-connection
 property that will also allow changing name servers' order in
 *resolv.conf*.

 Other planned features are a D-Bus API to expose current DNS
 configuration and integration with systemd-resolved.

## How it works

 On Linux and other Unix systems, applications use the name resolver
 provided by the C library, which reads the */etc/resolv.conf* file to
 know how to perform lookups. A typical *resolv.conf* could look like:

    nameserver 10.3.2.1
    nameserver 10.3.2.2
    domain mycompany.com

 which tells the resolver to contact server 10.3.2.1 first and if it's
  not reachable to try with 10.3.2.2, appending the suffix
  *.mycompany.com* to unqualified names (the ones without a dot).

 Enabling the system to properly resolve names is a certainly a duty
 of a network configuration daemon. NetworkManager gathers information
 about DNS servers by different means like DHCP, IPv6 SLAAC, IPCP (for
 PPP), VPNs or static configuration and can set up the system to use
 them according to several strategies, which are controlled by the
 *'dns'* option in *NetworkManager.conf*. By changing this option
 users can decide either to:

 * let NetworkManager update name servers and write other information
    to *resolv.conf*

 * start a local dnsmasq instance and make *resolv.conf* point to
    it. This also allows to perform *split DNS*, which means that the
    dnsmasq resolver can use different upstream servers depending on
    the domain being resolved. For example, queries for names
    belonging to the *.mycompany.com* domain could be directed to
    servers provided by the user's work VPN, while other names would
    use servers from the Internet provider

 * let NetworkManager interface with unbound and dnssec-triggerd,
    providing a split DNS configuration with DNSSEC support

 * have NetworkManager not manage *resolv.conf* and update it manually

The manual page for NetworkManager.conf is quite complete and
describes the usage of the mentioned *'dns'*, *'rc-manager'* options,
along with the syntax for specifying a static configuration. Help for
DNS options can instead be found in "man nm-settings". All
manual pages are also [available online][2].

[1]: https://mail.gnome.org/archives/networkmanager-list/2016-April/msg00064.html
[2]: https://developer.gnome.org/NetworkManager/1.2/manpages.html
