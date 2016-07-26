---
layout: post
title: "Hunting a segfault in Ruby on Rails"
date: 2016-07-25
tags:
  - ruby
  - rails
  - debugging
---

One of the benefits of modern languages is that you have some really great layers of abstraction between the programmer and the computer's hardware. Very rarely will most programmers have to directly manage memory, for example. This makes life so much easier! However, once in a while the scorching sun of reality burns through and forces us to confront the truth: abstractions hide the management, but its still there.

Much to my chagrin, I recently had deal with a stack trace starting with heart-wrenching declaration that it was a `segmentation fault`. Specifically:
`usr/local/lib/bundle/ruby/1.9.1/gems/activesupport-3.2.21/lib/active_support/cache.rb:561: [BUG] Segmentation fault`

## segfault?

Yes, segfault. This is an exception independent of the programming language used. It indicates that the application tried to access memory that illegal. This could be a read or write, and "illegal" could mean any number of things, usually that the address is outside of what the program has been allocated by the OS. In memory-managed languages like ruby, segfaults are relatively rare and generally relate to things like unsafe threading, serialization, or low-level networking.

## follow the code

Luckily the rails stack trace gives us a good starting point for hunting down the error. [The referenced line of the ActiveSupport cache](https://github.com/rails/rails/blob/0991c4c6fc0c04764f34c6b65a42adce190440c3/activerecord/lib/active_record/associations/association.rb#L156) is an attempt to marshal the object passed in. The rest of the stack trace indicates this is happening for new user creation. **BUT!** Attempting to run through the code in a rails console by pulling the exact same user out of the DB and marshalling it fails - or succeeds? It doesn't crash, which is a failure to reproduce. Attempting to log the user with a simple `puts user.to_json` seem to indicate it is exactly the same as when we pull it from the DB. Something funky is going on.

## breakpoints

Since we can't reproduce this by running through the code and everything we log looks correct, the next option is a live debugger. This is a actually a little tricky with rails; we can't easily run things locally since we need several external services active for this particular user creation flow. Instead, we can isolate a dev server and modify everything there. Normally all HTTP requests hit an ELB which routes to several EC2 instances. The instances use Apache to handle the traffic from the ELB and serve static assets, with multiple Rails processes loaded by [Phusion Passenger](https://www.phusionpassenger.com/) as an apache module. For debugging purposes we need to undo basically all of that and get the user creation request sent to a single Rails process running in a TTY.

The first step is to isolate the server so that only our test traffic goes to it. This involves 3 steps:

1. Remove the EC2 instance from the ELB
2. Create a record in route53 pointing to just the instance
3. Update the security group of the instance to allow port 80 (normally only allowed from the ELB)

After this, we can now hit the node directly and no other traffic is routed to it. Next, we need to modify apache to remove passenger and send everything straight to rails. The easiest way is to use mod_proxy and forward the requests to a local port:

```bash
# disable passenger
sudo a2dismod passenger
# enable mod_proxy
sudo a2enmod proxy
sudo a2enmod proxy_http
```

Then enable the proxy in the apache config:

```apache
# Keep requests for /static routed to local assets
ProxyPass /static !
# Send everything else to port 3000
ProxyPass /
ProxyPass / http://127.0.0.1:3000/
ProxyPassReverse / http://127.0.0.1:3000/
ProxyPreserveHost on
<Proxy *>
Order deny,allow
Allow from all
</Proxy>
```

Almost there! In the ruby code, right after user creation before caching, we add a breakpoint with the [debugger gem](https://github.com/cldwalker/debugger):

```ruby
require 'debugger'
debugger
```

We start the rails server in the foreground with `bundle exec rails s` and restart apache: `sudo service apache2 restart`. Now when we trigger the user creation, instead of everything exploding in our face we get a ruby console with the newly-created user object. Sweet!

## resolution

According to the docs, `Marshal.dump(user)` calls `user.marshal_dump`. Some [digging in ActiveRecord](https://github.com/rails/rails/blob/0991c4c6fc0c04764f34c6b65a42adce190440c3/activerecord/lib/active_record/associations/association.rb#L156) shows that the `marshal_dump` contains everthing returned by `instance_variables` except `:@reflection`. Since we have the troublesome user object in a live shell, we can run through all of those and compare them to the same user pulled from the database:

```ruby
user.instance_variables.each{|var| puts user.instance_variable_get(var) }
```

It quickly became obvious that the newly created user has associations attached to it, while the same user pulled from the DB does not. These associations are unable to be marshalled which leads to the segfault. After discussing this with some other members of the team, it was decided that marshal was massive overkill. We only care about the class of the object and the attributes, not all of the metadata. So the solution we went with is to change this

```ruby
Rails.cache.write(cache_key, user, expires: 1.day)
```

to

```ruby
Rails.cache.write(cache_key, user.to_yaml, expires: 1.day)
```

along with the corresponding reads. Hard to track down, super easy fix. Oh, and we burned that dev box to the ground and created a new one to replace it. Can't go around [mutating state](http://martinfowler.com/bliki/ImmutableServer.html) after all.
