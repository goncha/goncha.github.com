---
layout: post
title: "Disable IPv6 in Debian"
tags: debian network
---

Disable IPv6 function in Debain,

    sh# echo net.ipv6.conf.all.disable_ipv6=1 > /etc/sysctl.d/disableipv6.conf

Then reboot.