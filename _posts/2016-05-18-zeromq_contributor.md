---
layout: post
title: My 2016 GSoC Project - Part IIa
subtitle: Contributing to ZeroMQ
category: Programming
tags: [GSoC, ZeroMQ]
--- 

In the previous post I mentioned:

>"[...] However, GSoC is about learning new things, possibly how to do something better, and I should be willing to open my mind to different approaches for a problem (If in the end it doesn't convince me, maybe I could try contributing code and ideas to ZeroMQ itself, lol)."

Well, looks like it came sooner than expected:


As awesome as **ZeroMQ** is, something about it that instantly bothered me is the lack of [UDP](https://en.wikipedia.org/wiki/User_Datagram_Protocol) support. For applications where it would make sense to use **ZeroMQ** and low latency is a greater concern than reliability, like some which I have, that option is really missed.

Again, I'm not the only one thinking that way, and **UDP** support [had been request for a long time](https://github.com/zeromq/libzmq/issues/807). But recently one of the main developers, **Doron Somech** or **somdoron** on **Github** (great guy, by the way), added **UDP** for one of the new **ZeroMQ** communication patterns, **Radio-Dish** (like a thread-safe **Publisher-Subscriber**), to be available in the forthcoming release (**4.2.0**).

The "problem" with **Radio-Dish** is that it expects you to use the **ZeroMQ** message format, so you won't be able to communicate with applications that use **BSD sockets** directly without modifications on them. So, as naive and arrogant as I am*, the one who writes tried to become an open source contributor by submitting [a patch to enable UDP support for the Client-Server pattern](https://github.com/zeromq/libzmq/pull/1936) as well, taking advantage of the underlying infrastructure that **somdoron** set up for his work.

As it turns out, the effort to make **UDP** connections provide the same features that other protocols allow for **Client-Server** was bigger than I expected, and I was convinced that it wouldn't be a good idea for now.

But... But... I do really want to use **UDP**... (*insert river crying here*)

After the reality check, I noticed the **ZMQ_STREAM** option that I mentioned on my last post, exactly intended for communicating with conventional **TCP** clients and servers. 

So I thought, stubborn as always: "Why not **UDP** then?".

After a little more careful coding, [I submitted a pull request for RAW Datagram (UDP) sockets](https://github.com/zeromq/libzmq/pull/1986), under the option **ZMQ_DGRAM**. The developers actually got interested in the idea, and after a long (in the good sense of the word) exchange of opinions, which helped me to improve the patch and (try) to make it work, while learning a lot about **ZeroMQ** internals.

In the end, my **PR** itself wasn't merged, but **somdoron** offered to take my code, fix it and push it to the main tree (have I already mentioned how nice he is ?). Besides, for this partial contribution, [my name was added to the project's **AUTHORS** file](https://github.com/zeromq/libzmq/commit/5f0ac2aebe5f0633c93244ca5c26f54d8e5d17f0), so now it's official: **Behold, mortals, I'm a contributor**.


I know that it looks kinda lame from the outside (and that I should be coding for **Scilab**, not **ZeroMQ** !). This is not the type of thing that would make you popular at a party, right ?

Even so, I do feel really proud for being able to provide something (as little as it is) to the community.


**My self depreciating comments tend to be 50% joke and 50% truth*
