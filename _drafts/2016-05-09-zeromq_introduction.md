---
layout: post
title: My 2016 GSoC Project - Part II
subtitle: "ZeroMQ: The Swiss Knife of distributed computing"
category: Programming
tags: [GSoC, Scilab, ZeroMQ]
--- 

Hello again, fellow readers. Today I'll cover what are basically the building blocks of my future work: the [ZeroMQ](http://zeromq.org/) sockets.

As I briefly commented on my previous post, [Jupyter](http://jupyter.org/) works through the interprocess communication between multiple frontends and backends, enabling access to different programming languages from the same user interface. Let's recap:

![Messaging](https://jupyter-client.readthedocs.org/en/latest/_images/frontend-kernel.png)

(The example above shows the [IPython](https://ipython.org/) kernel, **Jupyter** backend implementation for the always nice [Python](https://www.python.org/) language [if you didn't know it, I really recommend that you learn about it]. That happens because **IPython** is the reference implementation [actually, **Jupyter** came from an split in the **IPython** project], but the same applies to any other language backend.)

So, these communications are performed through **ZeroMQ** connections. 

"But what is it, exactly ?"

Well, I could try to explain this myself, but I'll show you the "In a Hundred Words" definition from the [ZGuide](http://zguide.zeromq.org/page:all):

>ZeroMQ (also known as ØMQ, 0MQ, or zmq) looks like an embeddable networking library but acts like a concurrency framework. It gives you sockets that carry atomic messages across various transports like in-process, inter-process, TCP, and multicast. You can connect sockets N-to-N with patterns like fan-out, pub-sub, task distribution, and request-reply. It's fast enough to be the fabric for clustered products. Its asynchronous I/O model gives you scalable multicore applications, built as asynchronous message-processing tasks. It has a score of language APIs and runs on most operating systems. ZeroMQ is from iMatix and is LGPLv3 open source.


