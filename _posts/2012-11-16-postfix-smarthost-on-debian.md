---
layout: post
title: "Postfix Smarthost on Debian"
tags: postfix debian
---

When testing mail notification of Nagios, I choosed to relay mail from Nagios host (Debian 6) to a SMTP server, which can relay mail to internet. After Postfix be installed on Naigos host and configured with the option *Stallite system*, Nagios host can only relay mail to internet but not to local user. Solved problem after searching web, configured Postfix with the option *Internet with smarthost* fixing the problem. The following is the diff of main.cf in different configuration:

    --- main.cf.bak 2012-11-16 15:02:51.000000000 +0800
    +++ main.cf     2012-11-16 15:05:59.000000000 +0800
    @@ -30,10 +30,10 @@
     mailbox_size_limit = 0
     recipient_delimiter = +
    -inet_interfaces = loopback-only
    +inet_interfaces = all
     inet_protocols = ipv4
