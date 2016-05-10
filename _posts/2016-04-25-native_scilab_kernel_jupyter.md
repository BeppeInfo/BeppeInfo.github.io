---
layout: post
title: My 2016 GSoC Project - Part I
subtitle: Jupyter, Scilab, and why Matlab kinda sucks
category: Programming
tags: [GSoC, Scilab, Jupyter]
--- 

(Before we start, I must say that, according to GSoC [calendar](https://developers.google.com/open-source/gsoc/timeline), the program is now at the [Community Bonding Period](http://googlesummerofcode.blogspot.com.br/2007/04/so-what-is-this-community-bonding-all.html), not the coding phase itself. So, I really should be talking to the devs to plan my work for the coming months instead of writing ego-feeding hype posts, lol.

Anyways, let's move on.)

As said on the previous blog post, this time I'm going to talk more specifically about my GSoC project, which is named "A Native Scilab Kernel for Jupyter".

"So, what is it ?"

Well, from my point of view, is a good opportunity to change the views of some of my colleagues.

"What ?"

![let_me_explain](/img/kevin-hart-let-me-explain.jpg)

At the Mechatronics Engineering course, a mathematical tool for simulation is more than mandatory. And that tool pretty much always ends up being Matlab.

Not that there is something wrong with Matlab itself (besides being a huge 10+ GB resource hungry blob, I guess). It has great toolboxes, integration with many other applications, professional grade documentation and learning material, and other things that you would expect from a commercial solution but that I don't know because I simply do not use it.

"Why ? Are you some sort of hipster ?"

A software hipster, maybe.

In my course there isn't that many software entusiasts like myself. People tend to use whatever works and is well established and get things done. And that practical posture (which is really positive sometimes) leaded to a culture of perpetuating the usage of Matlab for any task involving programming, calculations and data visualization.

I can't help but feel that people are learning to hammer nails with a damn big golden sledgehammer. My interest for software makes me very avid for finding an optimal (which I generally see as leaner) solution for computing works, and Matlab seems so full of dependencies that you pull just to plot a Bode diagram....

![overengineering](https://i.imgur.com/JPsizDt.jpg)

Engineering is all about applying "the right tool for the job", right ?

Beyond that, Matlab is an expensive software. That's OK if you can and are willing to pay, but most of my colleagues can't, and after their student license expires, many of them end up resorting to piracy to keep the habit. As someone who is so much into software idealism, at least relatively, this gives me a bad feeling as well.

Not that I think people should code everything in pure ANSI C89 (well, maybe I do), but why not use Python (+Numpy +Matplotlib) ? Why not Octave ? Why not... Scilab ? These tools have more than enough resources to handle 99% of graduation (or even graduate) students use cases. But we keep ourselves into the vicious cycle of cultivating bad practices.

"So, what could we do ?", you ask me (didn't you ?)

Cost can't be our selling point for new solutions, because people ignore cost. "Openness" ? Hmmm... I'm not sure this helps.

We need a killer feature. Something that Matlab doesn't naturally offer and looks **cool**. 

Like... this level of coolness: ![cool](https://s-media-cache-ak0.pinimg.com/736x/b5/5a/18/b55a1805f5650495a74202279036ecd2.jpg)

(OK, we don't need **THAT** much !)

But, y'know, there is this nice thing called [Jupyter](http://jupyter.org/)

![Jupyter_logo](http://jupyter.org/assets/main-logo.svg)

"Oh, man ! Look at th.... wait, what is it ?"

Basically (AFAIK), a way to use different interpreted programming languages (mainly mathematical, data science and scientific computing ones) from the same Shell/GUI, calling functions, visualizing data in text or graphical format, playing with interactive interfaces... everything that your language of choice offers.

"Well, couldn't I do this in a terminal with multiple tabs already ?"

I guess so... But can you do it remotely ?

"OMG ! (wait... ssh ?)"

At its core, Jupyter defines the messaging that needs to happen between the frontend (the web, desktop or mobile based interface you choose) and the backends (kernels where the processing of commands itself happen) for all the different language implementations. The communication should use that nice library called [ZeroMQ](http://zeromq.org/) that supports many types of IPC protocols, including TCP sockets, which allows frontend and backends to run on the same or distinct machines.

Suppose that you're tired of always carrying that huge gaming/workstation laptop to class, but always need to hack that quick script on your hungry Matlab instalation in order to visualize some results... What about changing it for using your smartphone/tablet to access a remote server that gets your code, does the heavy lifting, and gives you back the useful data ? 

Looks nice, I suppose. At least [Wolfram Alpha](https://www.wolframalpha.com/) made some success in my classroom with that sort of feature.

So, that's my job for this Summer (actually, Winter here in the South). If I achieve this (**I DO REALLY HOPE SO**), maybe the next Engineering students will have a nice and easy way of ditching Matlab for some use cases, and learning this great tool that Scilab is (as well as some other shiny new languages). 

If I got you interested to know more about it, here is some graphical and more technical explanation (from [Jupyter Docs](https://jupyter-client.readthedocs.org/en/latest/messaging.html)):

![Messaging](https://jupyter-client.readthedocs.org/en/latest/_images/frontend-kernel.png)

Now you can see that it's more complex that it seems. The kernel has to be able to handle multiple clients or frontends, multiplexing between them, and that management happens with the help of a number of distinct sockets and message types (each one with its defined structure).

Wow. I spent so much trying to joke around that got a little tired for writing more technical stuff. Not that I learned much yet, but I'll do it and keep you all aware of my achievements in the process.

Until next time.
