---
layout: post
title: "How to restrict network access of LXC container"
date: 2022-01-27 00:00:00
categories:
 - blog
tags:
 - linux
 - itsec
 - tutorial
---

So let's say you have to run that sketchy or untrusted executable/project/binary/whatever and you want to do in a safe manner. Wide range of applicable solutions based on virtualization and contenerization technologies allow secure examination of suspicious software in a sandboxed environment.

If you're a Linux user, you'd probably point your attention to [LXC\[0\]][0]. Just a few commands to bring up shiny new environment based on a distro selected from wide variety of available options, running on the host kernel with no excessive emulation overhead.

That alone provides decent separation from whatever you don't want an object of research to touch in your workstation. **But what about connectivity?**

<!--break-->

Permitting an unknown actor unrestricted access to the Internet or LAN poses serious threat. At the same time, the container is going to need it to some extent, for installing packages or remote access from host. Wouldn't it be convenient to switch the network access restrictions of the container on and off at any time?

## Use a NAT bridge

Fairly simple way to achieve it is to use **iptables** in conjunction with **lxc-net**. Using *lxc-net* assumes running the container behind a NAT bridge. If you want to bridge the container to LAN, the tool of your choice might provide a built-in mechanism to achieve similar effect[\[1\]][1]. In my opinion, having the container behind local NAT provides higher level of resilience and control. In case of having to expose the container to wider publicity, you can always redirect traffic from host to lxc.

```
# iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 2221 -j DNAT --to-destination 10.0.3.100:22
```

The example rule above forwards TCP traffic origination on port `2221` of `eth0` interface to lxc SSH server running on `10.0.3.100:22`[\[2\]][2].

Arch Linux Wiki [article\[2\]][2] I took this rule from serves as a pretty good introduction to using *lxc-net* and lxc as whole. There's no point in me repeating most of its content, so give it a read if you're not familiar with lxc. At the end of a day, you should have a lxc container running behind NAT managed by *lxc-net*.

## Firewall setup

Assuming that you have used default configuration, CIDR of your lxc subnet is `10.0.3.0/24`. Running `lxc-ls -f` returns a list of containers similar to this:

```
NAME    STATE   AUTOSTART GROUPS IPV4       IPV6 UNPRIVILEGED
lxc0    RUNNING 0         -      10.0.3.250 -    false
lxc1    STOPPED 0         -      -          -    false
```

We'll be imposing restrictions on `lxc0` container. Take note of its IP address – `10.0.3.250` in my case.

You should also know name of the interface managed by *lxc-net*. It's mentioned in the config file `/etc/lxc/default.conf` by `lxc.net.0.link` key and by default it's `lxcbr0`.

If you run `iptables -L FORWARD -v` you should see two rules related to it at top of the chain.

```
pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     all  --  any    lxcbr0  anywhere             anywhere
    0     0 ACCEPT     all  --  lxcbr0 any     anywhere             anywhere
```

We want to forbid ANY incoming and outgoing traffic from the container except for traffic between the container and host. The address of host can be obtained from `/etc/default/lxc-net` file under `LXC_ADDR` key and its default value is `10.0.3.1`.

Therefore, this rule should be applied on host *iptables*:
```
# iptables -I FORWARD 1 -i lxcbr0 ! -d 10.0.3.1 -j DROP
```

It can be explained in a following way:
- `-I FORWARD 1` inserts the rule at index 1 of `FORWARD` chain
- `-i lxcbr0` applies the rule to traffic received on interface called `lxcbr0`
- `! -d 10.0.3.1` applies the rule to traffic directed to any IP address, **except** for `10.0.3.1` which is host IP
- `-j DROP` blocks all the traffic matching criteria above

Additionally, you might block inbound traffic:
```
# iptables -I FORWARD 2 -i lxcbr0 ! -s 10.0.3.1 -j DROP
```

In order to test the solution, try connecting to the container's shell via SSH. It should work fine. Now try pinging any host inside or outside LAN. None of the packets should go through.

Restriction can be lifted at any time with this command:
```
# iptables -D FORWARD 1
```

It removes the rule from chain FORWARD specified by the index. **Note that you need to run it twice if you applied both inbound and outbound blocking rules.**

## Allowing multiple containers to communicate

If you want to run several containers in the same subnet and allow the traffic to occur between them, but not leak outside, you can modify the rule to drop anything except for packets sourced and addressed to `10.0.3.0/24` subnet.

This turns the first rule into:
```
# iptables -I FORWARD 1 -i lxcbr0 ! -d 10.0.3.0/24 -j DROP
```

Equivalent modification of `-d` argument can be applied to the outbound rule.

## Links
~~~
[0]: https://linuxcontainers.org/
[1]: https://wiki.archlinux.org/title/Network_bridge
[2]: https://wiki.archlinux.org/title/Linux_Containers#Example_iptables_rule
~~~

[0]: https://linuxcontainers.org/
[1]: https://wiki.archlinux.org/title/Network_bridge
[2]: https://wiki.archlinux.org/title/Linux_Containers#Example_iptables_rule