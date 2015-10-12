---
layout: post
title: Dynamic DNS Load Balancing
date: 2013-12-23 19:51:27
tags:
  - python
  - system administration
  - dns
  - devops
  - load balancing
---
Using a small Python script that runs when a new server is created lets us automatically add the server to DNS for round-robin based load balancing. This makes sure that new servers are immediately available for use without having to modify existing servers or code.

Note that in order for this method to work you need some way to dynamically obtain the type of service the new server is running. For our naming scheme, hosts are named _IDENTIFIER-SERVICE-ENVIRONMENT_, making it easy to parse out the service name.

As an example of what this looks like, let’s say I have a server called _cyberdyne-web-sfo-prod_ with an IP address of 13.37.1.1. After setting up the server, I run one of the scripts on it. I can now query my DNS server for either _web.skynet.hiveary.com_ or _cyberdyne-web-sfo-prod.skynet.hiveary.com_; both will resolve to 13.37.1.1.

I then setup another server called _techcom-web-sfo-prod_ with an address of 13.37.1.2 and run the same script on it. Now querying _techcom-web-sfo-prod.skynet.hiveary.com_ always resolves to 13.37.1.2 and _cyberdyne-web-sfo-prod.skynet.hiveary.com_ always resolves to 13.37.1.1; querying web.skynet.hiveary.com, however, will return **both** 13.37.1.1 and 13.37.1.2. The order in which they are returned is rotated in a round-robin fashion with each request.

The first step is to make sure that dynamic DNS is enabled for the zones that will be updated. For BIND9, this is done with the “allow-update” option (actual names and IPs changed to protect the innocent):

```
acl internal-ips {
  127.0.0.1;
   13.37/16;
 };
 zone "skynet.hiveary.com" IN {
   type master;
   file "skynet.hiveary.com.zone";
   allow-update { "internal-ips"; };
 };
 zone "37.13.in-addr.arpa" IN {
   type master;
   file "37.13.in-addr.arpa.zone";
   allow-update { "internal-ips"; };
 };
 ```

 Now the DNS server is setup for dynamic updates. On the server to be added, make sure Python is installed (both 2.x and 3.x work), and install the excellent [dnspython](http://www.dnspython.org/) package: `sudo pip install dnspython`

 The last step is to run the update script (linked from the end of this post) on the server using whatever your preferred configuration management method is. I’ve created two versions of a Python script for this purpose; the first uses the sockets library to discover host information, while the second is written to be used as a [Salt](http://www.saltstack.com/community/) template. Either script can be modified fairly easily to fit other systems.

---

Dynamic DNS Python scripts:
[Using sockets](https://github.com/hiveary/scripts/blob/master/dns/dynamic-update.py)
[Using salt templates](https://github.com/hiveary/scripts/blob/master/dns/salt-dynamic-update.py)
