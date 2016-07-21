---
layout: post
title: "Hunting a segfault in Ruby on Rails"
tags:
 - ruby
 - rails
 - debugging
---

remove from ELB, set temp DNS record pointing at instance, update security group to allow port 80 directly without going through ELB

```bash
sudo a2dismod passenger
sudo a2enmod proxy
sudo a2enmod proxy_http
```

apache config:
```
ProxyPass /static !
 ProxyPass /
 ProxyPass / http://127.0.0.1:3000/
ProxyPassReverse / http://127.0.0.1:3000/
ProxyPreserveHost on
<Proxy *>
Order deny,allow
Allow from all
</Proxy>
```

set breakpoint: `require 'debugger'; debugger`
start rails server locally `bundle exec rails s`

restart apache: `sudo service apache2 restart`
