---
layout: post
title: My 2017 GSoC Project - Part XIII
subtitle: Rough Edges of A Promising Piece
category: Programming
tags: [GSoC-2017, Scilab, Modelica]
---

Hello again,

The time has come and I can't go any further inside the ongoing **GSoC Final Term Evaluation** period. Therefore, I shall "lay down the pencil" and submit the work done so far.

Fortunately, I have some new positive results to share.

### Successes

At the time of [my last post]({% post_url 2017-08-15-new_breath %}), **segfaults** in **FMI2** calls during simulation were preventing us to have a usable **OpenModelica** based solution. Gladly, I was able to track down the problem to its root, a **fmi2Terminate** call, that turned out to be unneeded. That way, now we can run a complete **Modelica** diagram simulation without crashing **Scilab**.

Also, I discovered that some **OpenModelica** forum searches lead me to outdated information, and the default options for model library compilation on which I was relying have been changed. Now, for building a pure **Model Exchange** library with no dependencies, as originally intended, the generation call inside **.mos** scripts must receive a few extra arguments, like in:

{% highlight python %}
buildModelFMU( model_name, version="2.0", fmuType="me", platforms={"static"} );
{% endhighlight %}

Finally, comparison between the old **modelicac** compiler and **OpenModelica** one shows that the new implementation is able to yield similar results, which is a must, in most of the cases:

<div style="float:left; width:50%;">
  <p align="center">
    <img src="/img/modelicac_chaos.png">
  </p>
</div>
<p align="center">(<b>modelicac</b> "Chaos" model simulation)</p>
<div style="float:left; width:50%;">
  <p align="center">
    <img src="/img/omc_chaos.png">
  </p>
</div>
<p align="center">(<b>omc</b> "Chaos" model simulation)</p>

<div style="float:left; width:50%;">
  <p align="center">
    <img src="/img/modelicac_rlc.png">
  </p>
</div>
<p align="center">(<b>modelicac</b> RLC circuit simulation)</p>
<div style="float:left; width:50%;">
  <p align="center">
    <img src="/img/omc_rlc.png">
  </p>
</div>
<p align="center">(<b>omc</b> RLC circuit simulation)</p>

<div style="float:left; width:50%;">
  <p align="center">
    <img src="/img/modelicac_rot.png">
  </p>
</div>
<p align="center">(<b>modelicac</b> 2nd order model simulation)</p>
<div style="float:left; width:50%;">
  <p align="center">
    <img src="/img/omc_rot.png">
  </p>
</div>
<p align="center">(<b>omc</b> 2nd order model model simulation)</p>

<div style="float:left; width:50%;">
  <p align="center">
    <img src="/img/modelicac_ball.png">
  </p>
</div>
<p align="center">(<b>modelicac</b> bouncing ball simulation)</p>
<div style="float:left; width:50%;">
  <p align="center">
    <img src="/img/omc_ball.png">
  </p>
</div>
<p align="center">(<b>omc</b> bouncing ball simulation)</p>


### Persisting Issues

As seen in the last comparison, I still couldn't find a way to make the **FMI2** wrapper properly handle **events** such as mechanical collisions, even if they are at least correctly detected.

Another issue that prevented me from testing more **Xcos Modelica** examples was a bug in generation of **flattened** (all code combined into a single file) **.mo** files, where mathematical signs end up duplicated, causing errors in the following compilation, like in the one from **"Ball in a platform"** demo:

{% highlight python %}
class Ball_Platform_im
  parameter Real Ball_Platform1__g = 9.800000000000001;
  parameter Real Ball_Platform1__m1 = 0.5;
  parameter Real Ball_Platform1__m2 = 0.3;
  parameter Real Ball_Platform1__k = 2.0;
  Real Ball_Platform1__y1(start = 11.0);
  Real Ball_Platform1__v1(start = 0.0);
  Real Ball_Platform1__y2(start = 15.0);
  Real Ball_Platform1__v2(start = 1.0);
  Real Ball_Platform1__y0;
  discrete Real Ball_Platform1__v1p;
  discrete Real Ball_Platform1__v2p;
  output Real OutPutPort1__vo;
  Real OutPutPort1__vi;
  output Real OutPutPort2__vo;
  Real OutPutPort2__vi;
equation
  Ball_Platform1__y0 = 10.0;
  der(Ball_Platform1__y1) = Ball_Platform1__v1;
  // Minus sign following plus sign and not isolated by parentheses
  Ball_Platform1__m1 * der(Ball_Platform1__v1) = if noEvent(Ball_Platform1__v1 < 0.001) and noEvent(Ball_Platform1__v1 > -0.001) then 0.0 else Ball_Platform1__k * (Ball_Platform1__y0 - Ball_Platform1__y1) + -0.2 * Ball_Platform1__v1 - Ball_Platform1__m1 * Ball_Platform1__g;
  der(Ball_Platform1__y2) = Ball_Platform1__v2;
  der(Ball_Platform1__v2) = if noEvent(Ball_Platform1__v2 < 0.001) and noEvent(Ball_Platform1__v2 > -0.001) then 0.0 else -Ball_Platform1__g;
  when Ball_Platform1__y2 < Ball_Platform1__y1 then
    Ball_Platform1__v1p = (Ball_Platform1__m1 * Ball_Platform1__v1 + Ball_Platform1__m2 * (2.0 * Ball_Platform1__v2 - Ball_Platform1__v1)) / (Ball_Platform1__m1 + Ball_Platform1__m2);
    Ball_Platform1__v2p = (Ball_Platform1__m2 * Ball_Platform1__v2 + Ball_Platform1__m1 * (2.0 * Ball_Platform1__v1 - Ball_Platform1__v2)) / (Ball_Platform1__m1 + Ball_Platform1__m2);
    reinit(Ball_Platform1__v1, 0.98 * Ball_Platform1__v1p);
    reinit(Ball_Platform1__v2, 0.98 * Ball_Platform1__v2p);
  end when;
  OutPutPort1__vi = OutPutPort1__vo;
  OutPutPort2__vi = OutPutPort2__vo;
  Ball_Platform1__y2 = OutPutPort1__vi;
  Ball_Platform1__y1 = OutPutPort2__vi;
end Ball_Platform_im;
{% endhighlight %}

As the error is caused by a **third-party** tool, **OMCompiler**, I've submitted a [**bug report**](https://trac.openmodelica.org/OpenModelica/ticket/4503#ticket) for it to be fixed **upstream**.

### Missing work

Apart from necessary fixes, remaining work for offering a full **modelicac** replacement involves improving **Windows** (mainly **64 bits**) compatibility (the **FMI2** library imports are formed erratically, most probably a upstream problem as well) and completing the integration of **OMCompiler** into **Scilab**'s build process.


### Final Thoughts

In the end, some aspects of this project ended up being more troublesome than expected, even if I knew since the beginning that it was a difficult task, and I couldn't make as much progress as I wanted. Nevertheless, I also admit lack of organization with my time and academic/real life affairs that held me back quite a lot. Shame on me, right ?!

On the other hand, I'm happy with all the learning experience and the fact that I was able to get the **OpenModelica** integration to work, to some extent. The **FMI2** specification, particularly, was a great finding that I surely intend to use for some of my future projects.

From now on it's all about cleaning up/submitting everything and waiting for my mentors evaluation. I'm thankful for all the help they provided me and I'll respect any decision regarding this final phase.

That's it for this **2017 GSoC**. Thanks one more time for sticking with me over the course of this (northern) summer. 

I hope to come back soon with new things to show.
