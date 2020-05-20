---
layout: post
title: "Routing some local hosts over a WireGuard VPN on an OpenBSD router"
---

> WireGuard kernel support on OpenBSD is coming (hopefully) soon! It's not in the `-current` branch or snapshots quite yet. [The repository where it's being worked on is here](https://git.zx2c4.com/wireguard-openbsd/). Once kernel support is in the `-current` branch of OpenBSD, I'll try to publish an updated guide using the kernel support instead of `wireguard-go`.

# Overview

I was inspired to write this by a friend who wanted to be able to automatically tunnel traffic from specific devices on their local network over a WireGuard VPN. A few other resources were helpful in getting up to speed on WireGuard and OpenBSD routing:

* [toying with wireguard on openbsd](https://flak.tedunangst.com/post/toying-with-wireguard-on-openbsd)
* [WireGuard on OpenBSD](https://blog.jasper.la/wireguard-on-openbsd.html)
* [OpenBSD Router : VPN](https://lipidity.com/openbsd/wireguard/)
* [`pf.conf` man page](https://man.openbsd.org/OpenBSD-6.6/pf.conf.5)
* [`rdomain` man page](https://man.openbsd.org/OpenBSD-6.6/rdomain.4)

# Prerequisites

You need a WireGuard configuration file from your VPN provider. For example, [Mullvad has a page to generate a config file here](https://mullvad.net/en/download/wireguard-config/). [There is an interesting post here about why Mullvad supports WireGuard](https://mullvad.net/en/blog/2017/9/27/wireguard-future/).

You also need root access to an OpenBSD router/firewall with PF running. It shouldn't matter if the OpenBSD system is directly connected to the internet or not. These steps were tested successfully on version 6.6 of OpenBSD, and they should work on 6.7 as well. All commands should be run as root (or with `doas`).

# Install WireGuard

This will install WireGuard from packages. [`wireguard-go` is a userland implementation of WireGuard written in Go](https://git.zx2c4.com/wireguard-go/about/). [`wireguard-tools` provides the `wg` and `wg-quick` command-line tools](https://git.zx2c4.com/wireguard-tools/about/). These steps will only use the `wireguard_go` daemon and the `wg` command.

```
pkg_add wireguard-go wireguard-tools
```

# Configure the service

Now you can set the `tun` interface you want WireGuard to use (we'll use `tun2` just because it's available; you can use any unused `tun` device number you want, but the rest of the steps here will assume you're using `tun2`) and enable it so it will start on boot.

```
rcctl set wireguard_go flags tun2
rcctl enable wireguard_go
```

# Set up the config file

This will put your WireGuard config file into `/etc/wireguard` and recursively set permissions that allow the new user you created to access it. This assumes the config file is named `mullvad-se12.conf` and is in your current working directory (you can copy any config file from wherever you downloaded it to).

```
mkdir /etc/wireguard
cp mullvad-se12.conf /etc/wireguard/client.conf
chown -R root:wheel /etc/wireguard
chmod -R 600 /etc/wireguard
```

For the config file to work with `wg` (instead of `wg-quick`, which doesn't allow for the level of customization we're doing here) **you have to delete both the `Address` and `DNS` lines from the `Interface` section**. Copy these lines somewhere else for now since you'll need the values later.

# Set up the network interface

Create the file `/etc/hostname.tun2` with the following contents. Replace `10.32.4.3` with the IPv4 address from the `Address` line that you previously deleted from `/etc/wireguard/client.conf` (note that the `/32` in the IPv4 address is replaced with the equivalent in dot-decimal notation).

```
description "WireGuard"
inet 10.32.4.3 255.255.255.255
mtu 1420
up
```

Bring up the interface.

```
sh /etc/netstart tun2
```

Apply your Wireguard configuration to the interface.

```
wg setconf tun2 /etc/wireguard/client.conf
```

# Configure a routing table

This will configure routes in an alternate routing table that can be used for the traffic we want to be routed over the VPN. I initially also put the `tun2` interface in an alternate `rdomain`, which worked fine, but I don't think it's strictly required here and we can get away with just a dedicated routing table (we'll use PF later to select the routing table for specific incoming traffic).

We'll use an ID of `2` for the routing table but you can use any value up to `255` (the default routing table is ID `0`). Remember to replace `10.32.4.3` here with the actual address you configured in the `hostname.tun2` file!

```
route -T 2 -n add -inet default -iface 10.32.4.3
```

One more route is needed to ensure that traffic to the WireGuard VPN peer is not routed over the VPN. Replace `185.65.134.128` with the IP address from the `Endpoint` line in `/etc/wireguard/client.conf` (make sure to remove the `:51820` port specification) and replace `172.15.1.1` with your internet connection's default gateway. If your OpenBSD system is connected directly to the internet, this is the `gateway` value in the output of the `route -T 0 -n get default` command.

```
route -T 2 -n add 185.65.134.128 -gateway 172.15.1.1
```

# Configure PF

Add the following rules to your `pf.conf` file. The first rule will match any incoming traffic from the local device with IP address `10.0.0.100` and assign it to the alternate routing table you created. The second rule will then match outgoing traffic on the `tun2` interface and NAT it to the IP address of the interface. Replace `em1` with the name of the interface that is connected to your local network. You can also specify a range of IP addresses here, or adjust the rule to match all incoming traffic on an interface (e.g., a VLAN interface).

```
match in on em1 inet from 10.0.0.100 to any rtable 2
match out on tun2 inet from !(tun2:network) to any nat-to (tun2:0)
```

Save your changes and reload your ruleset to apply the changes.

```
pfctl -f /etc/pf.conf
```

Your local device(s) should have all traffic routed over the WireGuard VPN now!

# DNS

**Nothing in these instructions covers DNS configuration to prevent DNS leaks!** [Mullvad does hijack all DNS requests and routes them to a Mullvad DNS server](https://mullvad.net/en/help/terms-service/). For good measure (or if you're not using Mullvad or another provider that does this automatically), you can manually set the DNS server on your local device, or configure your DHCP server to hand out the desired DNS server addresses to your local device or local subnet. For example, if your OpenBSD system is your DHCP server and you want to configure Mullvad's DNS server for the entire 10.0.1.0/24 subnet, include the following in `/etc/dhcpd.conf`.

```
subnet 10.0.1.0 netmask 255.255.255.0 {
    option routers 10.0.1.1;
    option domain-name-servers 193.138.218.74;
    range 10.0.1.100 10.0.1.199;
}
```

[The Mullvad DNS server address is available here](https://mullvad.net/en/help/dns-leaks/).

[If you use Firefox you may also want to disable DNS over HTTPS (DoH)](https://support.mozilla.org/en-US/kb/dns-over-https-doh-faqs#w_will-users-be-able-to-disable-doh).

# Running as a non-root user

> This is optional!

[WireGuard merged a patch in 2019 to support running as a non-root user](https://lists.zx2c4.com/pipermail/wireguard/2019-July/004308.html) if you set the MTU on the `tun` interface instead of relying on the WireGuard daemon to do it. Unfortunately even with this patch, you would still need to change the group ID (giving the group the daemon runs as `rw` permissions) on the `tun` device file in `/dev` for it to work. My knowledge of Go is extremely limited, but I think this is because the [`OpenFile` function](https://golang.org/pkg/os/#OpenFile) is called with `O_RDWR`, meaning it tries to open the file in read/write access mode. [You can see this on line 125 of `tun_openbsd.go`](https://git.zx2c4.com/wireguard-go/tree/tun/tun_openbsd.go?h=v0.0.20190908#n125) (this link points to the version of the file that corresponds with the version of the package available for OpenBSD 6.6, but the line is the same in more recent versions).

If you still want to set up WireGuard to run as a non-root user, add a user for it to run as.

```
useradd -s /sbin/nologin -c "WireGuard" -d /var/empty _wireguard
```

Configure the daemon to run as the user you just created.

```
rcctl set wireguard_go user _wireguard
```

Set permissions on your configuration files and `/dev/tun2`.

```
chown -R _wireguard:_wireguard /etc/wireguard
chmod -R 600 /etc/wireguard
chown root:_wireguard /dev/tun2
chmod 660 /dev/tun2
```

Restart the daemon.

```
rcctl restart wireguard_go
```

# Thanks!

Thanks for reading. This was a fun exercise in learning about WireGuard for me and I hope it's helpful for someone else. Stay tuned for an updated post using the OpenBSD kernel support when it's available.
