---
layout: post
title: My 2017 GSoC Project - Part III
subtitle: Computational Black Boxes
category: Programming
tags: [GSoC-2017, Scilab, Modelica]
--- 

Hey everyone,

I should have started posting updates for **GSoC Working Period** a week ago, but some difficulties in grasping how to do things properly, along with some of my academic research tasks, prevented me from having something worth publishing. I apologize for that.

On the bright side, when I decided to look for some material about **Xcos blocks** (what I should have done from the beginning), instead of trying to figure out everything by looking at the code, I quickly stumbled upon [this document](http://www.scicos.org/Newblock.pdf) detailing pretty much anything I needed on **Scilab**'s side.

### Scilab/Xcos block structure

As it turns out, the way each type of **Xcos** block operates is basically defined by 2 functions: a **interfacing** and a **computational** one. Both implement their respective standardized interfaces, in a way that the **simulator** handles any block as a **black box** with inputs and outputs, without being aware of its internals.

The **interfacing function**, always written in **Scilab** language, builds the **pop-up window** listing the modifiable properties for a given block. It also acts as a glue between generic **simulation** (time updates and numerical solvers) code and the specific **computation** code for that block. Its implementation for **OpenModelica** integration will be covered in more detail in the future.

<p align="center">
  <img src="/img/teleop_simulator_delay_config.png">
</p>
<p align="center">
  (Properties window for "Random Delay" Xcos block)
</p>

**Computational funtions** could be written in either **C** or **Scilab** language, which defines the block type (4 or 5, respectively, with lower values being used for legacy block types). In the case of **Modelica** blocks, as there is no **Scilab** code generation available for **OMCompiler**, we must implement a **C** function with signature like:

{% highlight cpp %}
void sim_func( scicos_block* block, int flag );
{% endhighlight %}

Where **scicos_block** is a generic block data structure type, declared in the header **scicos_block4.h**:

{% highlight cpp %}
typedef struct
{
  int nevprt;     // Binary coding of activation inputs, -1 for internal activation
  voidg funpt;    // Pointer to the computational function
  int type;       // Type of the computational function, in this case type 4
  int scsptr;     // Not used for C programming
  int nz;         // Size of discrete-time state vector
  double* z;      // Vector of discrete-time state
  int noz;        // Number of object states
  int* ozsz;      // Vector of sizes of object states
  int* oztyp;     // Vector of data types of object states
  void** ozptr;   // Table of pointers to object states
  int nx;         // Size of continuous-time state vector
  double* x;      // Vector of continuous-time state
  double* xd;     // Vector of the derivative of the continuous-state, same size nx
  double* res;    // Only used for internally implicit blocks, vector of size nx
  int nin;        // Number of regular input ports
  int* insz;      // Vector of sizes of regular input ports
  void** inptr;   // Tables of pointer to the regular input ports
  int nout;       // Number of regular output ports
  int* outsz;     // Vector of sizes of regular output ports
  void** outptr;  // Tables of pointers to the regular output ports
  int nevout;     // Number of event output ports
  double* evout;  // Delay time of output events
  int nrpar;      // Size of real parameters vector
  double* rpar;   // Vector of real parameters
  int nipar;      // Size of integer parameters vector
  int* ipar;      // Vector of integer parameters
  int nopar;      // Number of object parameters
  int* oparsz;    // Vector of sizes of object parameters
  int* opartyp;   // Vector of data types of object parameters
  void** oparptr; // Table of pointers to the object parameters
  int ng;         // Size of zero-crossing surfaces vector
  double* g;      // Vector of zero-crossing surfaces
  int ztyp;       // Boolean, True only if the block may have zero-crossing surfaces
  int* jroot;     // Vector of size ng indicates the presence and the direction of the crossing
  char* label;    // Block label
  void** work;    // Table of pointers to the block workspace (if allocation done by the block)
  int nmode;      // Size of modes vector
  int* mode;      // Vector of modes
}
scicos_block;
{% endhighlight %}

The integer **flag** informs the current calculation stage, as our function can be called many times during a single simulator iteration, for different purposes. Each stage flag value is indicative which fields from the **scicos_block** parameter should be considered as inputs, and how the function is expected to fill the outputs:

<table style="width:100%">
  <tr> <th>flags</th> <th>inputs                                </th> <th>  outputs   </th> <th>description                                    </th> </tr>
  <tr> <td>  0  </td> <td>t, nevprt, x, z, inptr, mode, phase   </td> <td>  xd, res   </td> <td>compute the derivative of continuous time state</td> </tr>
  <tr> <td>  1  </td> <td>t, nevprt, x, z, inptr, mode, phase   </td> <td>  outptr    </td> <td>compute the outputs of the block               </td> </tr>
  <tr> <td>  2  </td> <td>t, nevprt>0, x, z, inptr              </td> <td>   x, z     </td> <td>update states due to external activation       </td> </tr>
  <tr> <td>  2  </td> <td>t, nevprt=-1, x, z, inptr, jroot      </td> <td>   x, z     </td> <td>update states due to internal zero-crossing    </td> </tr>
  <tr> <td>  3  </td> <td>t, x, z, inptr, jroot                 </td> <td>   evout    </td> <td>program activation output delay times          </td> </tr>
  <tr> <td>  4  </td> <td>t, x, z                               </td> <td>x, z, outptr</td> <td>initialize states and other initializations    </td> </tr>
  <tr> <td>  5  </td> <td>x, z, inptr                           </td> <td>x, z, outptr</td> <td>final call to block for ending the simulation  </td> </tr>
  <tr> <td>  6  </td> <td>t, nevprt, x, z, inptr, mode, phase   </td> <td>x, z, outptr</td> <td>reinitialization (if needed)                   </td> </tr>
  <tr> <td>  7  </td> <td>                                      </td> <td>            </td> <td>only used for internally implicit blocks       </td> </tr>
  <tr> <td>  9  </td> <td>t, phase=1, nevprt, x, z, inptr       </td> <td>  g, mode   </td> <td>compute zero-crossing surfaces and set modes   </td> </tr>
  <tr> <td>  9  </td> <td>t, phase=2, nevprt, x, z, inptr       </td> <td>     g      </td> <td>compute zero-crossing surfaces                 </td> </tr>
  <tr> <td> 10  </td> <td>t, nevprt, x, z, inptr, mode, phase   </td> <td>    res     </td> <td>compute jacobian matrix                        </td> </tr>
</table>

### Hooking things up

Currently in **Scilab**, its customized **modelicac** compiler is able to generate an implementation of the **computational function** directly from **Modelica** code. For **OMCompiler**, one is only able to generate a **FMI2** interface implementation, so some generic wrapper code is needed.

From now on, I plan to (as soon as possible) start writing this wrapper and showing you how **FMI2** calls are performed for each one of the simulation stages.


Thanks for sticking with me one more time. See Ya !
