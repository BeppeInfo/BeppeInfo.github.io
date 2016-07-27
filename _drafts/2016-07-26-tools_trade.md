---
layout: post
title: My 2016 GSoC Project - Part XII
subtitle: Tools of The Trade
category: Programming
tags: [GSoC, Scilab, C++, Autotools]
---     

Hello, again (and quicker than last time) !

When it comes to solving a problem, much is said about the value of knowing precisely which screws to twist, rather than being capable of twisting a lot of screws. But what we normally neglect is that first you need to know how to use a **screwdriver** ! 

Well, it seems like that is a good analogy for software development of a certain magnitude.

**Scilab** surely is a complex project, and even the tools used to automate much of its code development, maintenance and compilation also end up being of a little complex usage. 

At least at first sight.

If you are somehow familiar with **Linux** and building programs from source, you most probably already faced this [maybe infamous] installation procedure on your **terminal**:

{% highlight bash %}
$ ./configure
...
$ make
...
$ sudo make install
...
{% endhighlight %}

