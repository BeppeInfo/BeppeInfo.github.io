---
layout: post
title: My 2017 GSoC Project - Part I
subtitle: Modeling with numbers
category: Programming
tags: [GSoC-2017, Scilab, Modelica]
---

Hi there,

As said on [my previous blog post]({% post_url 2017-05-12-google_summer_of_code_2 %}), today I'm talking more specifically about my **GSoC 2017** project, named **"OpenModelica Integration [for Scilab]"**.

*"So, what is this you plan to integrate ?"*

Imagine that you need to simulate a physical phenomenon, or implement any sort of equation, in your preferred C-like computer language (e.g. **C++** or **Python**)... Simply describing it as

<p align="center">
  <b>a = b * c + d</b>
</p>

only allows you to obtain **a**, given all the other values, right ? In this format, you couldn't calculate **c**, for instance, just by having **a**, **b** and **d**. If you need to obtain different variables at different processing steps, the equation must be arranged in all correspondent ways:

<table style="width:100%">
  <tr>
    <th>1) a = b * c + d</th>
    <th>2) b = (a - d) / c</th>
    <th>3) c = (a - d) / b</th> 
    <th>4) d = a - b * c</th>
  </tr>
</table>

That's called **causal modeling**: one have to describe every needed different relation between these terms.

When you think about it... It kind sucks for those types of problems, am I right ?

*"But what can we do about it ?"*

I'm glad you asked that (maybe not, but...). Thankfully, some people have already come up with solutions for **acausal modeling**, where you just define equations in one way and the other are implicitly derived from that. One of the most well-known tools for that is called [**Modelica**](https://www.modelica.org/).

<p align="center">
  <img src="https://upload.wikimedia.org/wikipedia/en/0/07/Modelica.png">
</p>

Not really a tool, but a standard. Have I ever said I love standards ?

Anyways, **Modelica** is an object-oriented and declarative language specification for description/modeling of dynamic systems (mechanical, electrical, thermal, mixed, etc.). Being domain neutral, different compiler and simulation engine implementations use the information written in a **Modelica** file (**.mo**) to generate source code for multiple programming languages (**C**, **C++**, **C#**, **Java**, etc.), used for actual processing.

<p align="center">
  <img src="/img/modelica_dc_motor.jpg">
</p>
<p align="center">
  (<b>Modelica</b> text and graphical (blocks) representations of an electromechanical system)
</p>

Currently, there are a lot of actual simulation tools (commercial and free) which implement the **Modelica** text and graphical languages, like **CATIA Systems**, **Dymola**, **LMS AMESim**, **JModelica**, **MapleSim**, **OpenModelica**, **SimulationX**, **Wolfram** and, guess what, **Scilab**.

In **Scilab**, **Modelica** language is used to describe more complex **Xcos** blocks (mainly from [**Coselica**](https://atoms.scilab.org/toolboxes/coselica) package), taking advantage of existing compiler/translator implementations to generate correspondent **C** code, in order to integrate those components into complex **Xcos** simulations. 

I have even used that in one of my academical works:

<p align="center">
  <img src="/img/teleop_simulator.png">
</p>
<p align="center">
  (Can you notice the green blocks ? They are <b>Modelica</b> blocks integrated into a <b>Xcos</b> model)
</p>

Nowadays, **Modelica** in **Scilab** is managed by an old **LMS** compiler, released under the **GPL** license, which is not actively maintained (needing out-of tree patches) and support only a subset of the current language specification. To address that, **Scilab** developers proposed modifications to allow the usage of [**OpenModelica**](https://www.openmodelica.org/)’s **OMCompiler** (**omc**) as an alternative. As a more up-to-date solution, **omc** integration would provide **Scilab**/**Xcos** the ability to use more advanced **Modelica** features, and avoid bugs, issues and maintenance burden of the current **modelicac** compiler.

That represents a pretty important task, but someone needs to step up in order to make it happen... Guess who is willing to try ?

The following months are going to be a tough test for my software development skills. I hope to succeed to some extent.

Next time, I'll bring you something more detailed about how **OMCompiler** works.


Thanks for sticking with me. See ya !
