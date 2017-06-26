---
layout: post
title: My 2017 GSoC Project - Part II
subtitle: One Interface to Rule Them All
category: Programming
tags: [GSoC-2017, Scilab, Modelica]
--- 

Hi all,

I was planning to write the next blog post only after I figured out completely how **Scilab** integrates with **Modelica** as of now. However, exploring all the necessary code paths was taking more time than expected, and I would like to conclude the [Community Bonding Period](http://googlesummerofcode.blogspot.com.br/2007/04/so-what-is-this-community-bonding-all.html) with a new update.

So I'll skip that a little bit and, as promised, get into more detail about the integration possibilities for **OpenModelica's OMCompliler**.

If you remember well, It's a know fact that I have a soft spot for standards... Well, at least the ones which don't fall into this category:

<p align="center">
  <img src="https://imgs.xkcd.com/comics/standards.png">
</p>
<p align="center">
  (Mandatory <a href="https://xkcd.com/">XKCD</a> reference)
</p>

"Competition breeds excellence", but sometimes reaching a consensus about a general way to do things, leaving room for evolution and specifics variation, sounds like a good thing. 

*But why are you taking about it... again ? Isn't **Modelica** or standard already ?*

Yes, but not only that.

### The Functional Mock-up Interface

Typical **Modelica** applications are useful to perform simulations for a given dynamic model, but usually one doesn't want to stop there, and wishes to use results from the same calculations in e.g. real-time control. Having to reimplement all mathematical expressions for the final use case demands additional coding and review work to ensure that both models (**Modelica** application's and hand coded one) behaviour match exactly.

*Wouldn't be great having the ability to export the original model in a self-contained format, easily integratable with target applications ?*

Well, some **Modelica** software do precisely that.

*Neat ! Wouldn't be also great if there is and standardized way to do it, so that you could switch between runtimes exported by different programs, and choose the one that works best for you ?*

Well, there is. It's called [Functional Mock-up Interface (FMI) specification](https://svn.modelica.org/fmi/branches/public/specifications/v2.0/FMI_for_ModelExchange_and_CoSimulation_v2.0.pdf) (currently at version **2.0**), for generation of interchangeable **C** code.

Gladly, **OpenModelica**'s **OMCompiler** supports it, allowing input **.mo** files to be translated to, among other formats, **Functional Mock-up** (**FMU**) packages, which include source code and binary for a library implementing processing logic and exposing a **FMI2** interface, as well as a model description **XML** file. 

Considering a **Model.mo** input file, one of the simplest ways to generate **FMI2**-compliant code would be running the command:

>$OMCOMPILER_PATH/omc Model.mo --simCodeTarget=sfmi --simulationCg

that generates files **Model_FMI.cpp** (ready to be compiled to a library or directly added to main code, along with **FMI2** headers) and **modelDescription.xml**.

In addition, **FMI2** specification defines 2 exporting modes for packages: **Co-Simulation** (Fig. 1,2), which includes code for model simulation solver/integrator, and **Model-Exchange** (Fig. 3,4), which ships just the bare minimum model code, leaving integration processing to the calling application.

<p align="center">
  <img src="/img/fmi2_cosimulation.png">
</p>
<p align="center">
  (Co-simulation library interface diagram)
</p>

<p align="center">
  <img src="/img/StateMachineCoSimulation.png">
</p>
<p align="center">
  (Co-simulation library operation state-machine)
</p>

<p align="center">
  <img src="/img/fmi2_model_exchange.png">
</p>
<p align="center">
  (Model Exchange library interface diagram)
</p>

<p align="center">
  <img src="/img/StateMachineModelExchange.png">
</p>
<p align="center">
  (Model Exchange library operation state-machine)
</p>

At first, it seems like a **Co-simulation** model is always prefereable, because of its simpler interface. However, its integration methods are limited to basic ones, and as **Scilab**/**Xcos** already uses [ODE](https://en.wikipedia.org/wiki/Ordinary_differential_equation) solvers (like [Sundials IDA](https://computation.llnl.gov/projects/sundials/ida)) during simulations, the **Model Exchange** model becomes more appropriate, because solvers can be shared consistently and changed dynamically across all model blocks. 

Current **Scilab**'s **modelicac** compiler works with its own interface, and **Xcos** blocks have incomplete support to **FMI** calls, so they'll need to be modified. The extra work may be worth it, though, as adopting **FMI2** for setting and getting model variables within **Xcos** environment has the benefit of allowing, in future, different **Modelica** implementations with **FMI** support to be easily integrated as well. Maybe even current **Modelica** implementation could be adapted to use the **FMI2 API** entirely, reducing the amount of code to be maintained.


That's it for now, folks. I don't know if I was careless and didn't interact properly with my mentors during **Community Bonding**, so I hope that doesn't prevent me for being approved in this first phase of the program, and we are able to continue this project until its completion.


See Ya !
