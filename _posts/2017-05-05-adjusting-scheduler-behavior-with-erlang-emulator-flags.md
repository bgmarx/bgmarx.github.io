---
layout: post
title: Adjusting Erlang Scheduler Behavior
tags:
- Elixir
- Schedulers
- performance
- Erlang
---

The subtitle of the post might as well: "but you shouldn't." Unless you know the inner workings of the BEAM, attempts to fine tune the virtual machine will almost surely backfire.

Why? It's pretty finely tuned already. Ericsson created Erlang to solve the rigorous demands of a telecom. Telecoms are heavily regulated because they're always expected to work. How many times in your life have you picked up a phone and not heard a dial tone (or now, I suppose not been able to make or receive a call)?

At Bleacher Report, for instance, we've not had to tune the VM at all. We experimented with some of the following changes but nothing out-performed the Erlang defaults. And in some instances performance appeared to be better but under heavy load, became much worse.

So why bother? Understanding the various changes one can make to the VM can help provides a greater understanding of how the VM works and in the case that you find you have a bottleneck, you can use that knowledge to wriggle your way out of it.


### Emulator Flag Options

The [default option](http://erlang.org/doc/man/erl.html) is `u` which stands for `unbound`. The schedulers are not bound to any core, and it's up to the discretion of the OS "where the scheduler threads execute, and when to migrate them." Presumably, this means that this behavior can vary on different OSes. 

Speaking of different OSes, if you're on OSX or not "on newer Linux, Solaris, FreeBSD, and Windows systems," you can't test any of the scheduler flags---`+s`---locally.

When we tried out various scheduler flags, the thought was that if the OS chooses when and where to schedule tasks, there might be a lot of jumping around inefficiently. As the number of cores---and therefore schedulers---increase, there would only be more jumping around. The first flag that we tried was `ns`, or _no spread_. `ns` states that "scheduler are bound as close as possible in hardware." 

This seemed like an interesting approach. If schedulers are aligned to cores, perhaps we can get better performance. I suppose the key word here is _hardware_. When you're using a cloud provider, and you're running in a Docker container on a virtual server somewhere, how does this actually relate to the hardware? It's hard to say. 

But we gave it a shot anyway. All we had to do was update how we start our server by passing in `+sbt ns` as the example below shows.

````console 
elixir --erl "+sbt ns" --sname app --cookie cookie 
````

And it performed pretty well---better even---with load testing and diverting traffic from production to staging. So we rolled it out. And forgot about it for a while. Then during high traffic events we started seeing the run queue increase. It didn't increase by much but it did increase. As our traffic spikes got ever bigger, we noticed the run queue would continue to increase a bit. And our system didn't perform quite as well. 

We scoured the code for code that could account for this rising run queue because we'd forgotten about the scheduler flag changes. Eventually, someone remembered and, since we'd run out of other ideas, we removed the `+s` flag and deployed the changes. With the default scheduler settings restored, the system performed better during traffic spikes. 

### Defaults Matter (In Production)

As the docs succinctly put it, "[h]ow schedulers are bound matters." Or, in a less perfunctory manner, "we know better than you why the defaults are what they are." And that's almost certainly true. 

In production, it's a safe bet that the default settings will meet your needs. But in staging or anywhere else, there's almost no harm in fidgeting with the abundant settings in the Erlang emulator. If nothing else, you'll get a deeper understand for and a greater appreciation of the Erlang VM. 