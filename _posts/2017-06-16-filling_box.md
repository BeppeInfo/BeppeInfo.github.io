---
layout: post
title: My 2017 GSoC Project - Part IV
subtitle: (Un)Filling The Box
category: Programming
tags: [GSoC 2017, Scilab, Modelica]
--- 

Hello there,

I was undecided about where to begin the posts about glueing **FMI2** calls to the **Scicos Block** interface, mentioned in [the last post]({% post_url 2017-06-10-blocky_package %}), so I'll take the most obvious route and start from the beginning... 

... of block processing/simulation, I mean. And that is triggered by the **Initialization flag** (4):

<table style="width:100%">
  <tr> <th>flags</th> <th>                inputs                </th> <th>  outputs   </th> <th>               description                     </th> </tr>
  <tr> <td> ... </td> <td>                  ...                 </td> <td>    ...     </td> <td>                  ...                          </td> </tr>
  <tr> <td>  4  </td> <td>               t, x, z                </td> <td>x, z, outptr</td> <td>  initialize states and other initializations  </td> </tr>
  <tr> <td> ... </td> <td>                  ...                 </td> <td>    ...     </td> <td>                  ...                          </td> </tr>
</table>

The purpose of the **Initialization job** is more thoroughly described in [the available documentation](http://www.scicos.org/Newblock.pdf):

*"Initialization: At the begining of the simulation, this job is called under flag=4 to initialize continuous and discrete states (if necessary). It also initialize the output port of the block. This function is not used in all of the blocks, it is used for blocks that needed dynamically allocated memory (the allocation is done under this flag), for blocks that read and write data from files, for opening a file, or by the scope to initialize the graphics window."*

So, aside from **continuos/discrete state and output initialization** (in arrays **x**, **z** and **outptr**, respectively), this flag also indicates the moment when all **manual memory allocations** (if needed) should be performed for a given block.

That's our case, because **FMI2 API** involves allocation of **components** to hold internal state values.

*But, as **scicos_block** is the only data structure provided at the **computational function call**, where will they be stored ?*

Taking a look at **scicos_block4.h** header, it seems there is a field intended for this very role:

{% highlight cpp %}
typedef struct
{
  /* ... */
  int nz;         // Size of discrete-time state vector
  double* z;      // Vector of discrete-time state
  /* ... */
  int nx;         // Size of continuous-time state vector
  double* x;      // Vector of continuous-time state
  /* ... */
  int nout;       // Number of regular output ports
  int* outsz;     // Vector of sizes of regular output ports
  void** outptr;  // Tables of pointers to the regular output ports
  /* ... */
  void** work;    // Table of pointers to the block workspace (if allocation done by the block)
  /* ... */
}
scicos_block;
{% endhighlight %}

Then, assuming that the **work** pointer is the one to be used, we initialized the block inside our **Scicos**/**FMI2** wrapper:

{% highlight cpp %}
#include "scicos_block4.h"

#include "scicos.h"
#include "scicos_malloc.h"
#include "scicos_free.h"

#include "scicos.h"
#include "scicos_malloc.h"
#include "scicos_free.h"

#include "fmi2TypesPlatform.h"
#include "fmi2FunctionTypes.h"
#include "fmi2Functions.h"

/* ... */

// Simple placeholder logging message
void printLogMessage( fmi2ComponentEnvironment componentEnvironment, fmi2String instanceName, 
                      fmi2Status status, fmi2String category, fmi2String message, ... )
{
  va_list args;
  va_start( args, message );
  vprintf( (const char*) message, args );
  va_end( args );
}

// The computational function itself
void fmi2_block( scicos_block* block, const int flag )
{
  static fmi2Component fmi2Block;

  fmi2EventInfo eventInfo;

  switch( flag )
  {
    /* ... */
        
    case Initialization:          // Value 4, part of a enumeration defined in scicos_block4.h
    {
      // User provided functions for FMI2 data management
      const fmi2CallbackFunctions functions = { .logger = printLogMessage,
                                                .allocateMemory = scicos_malloc,
                                                .freeMemory = scicos_free, 
                                                .stepFinished = NULL,
                                                .componentEnvironment = NULL };
      
      // Store FMI2 component in the work field     
      block->work = fmi2Instantiate( block->label,          // Instance name 
                                     fmi2ModelExchange,     // Exported model type
                                     block->uid,            // Model GUID, have to be obtained
                                     "file:/data_path/",    // Optional FMU resources location
                                     &functions,            // User provided callbacks
                                     fmi2True,              // Interactive mode
                                     fmi2True );            // Logging On
      
      // Define simulation parameters
      fmi2SetupExperiment( (fmi2Component) block->work,     // FMI2 component   
                           fmi2False,                       // Tolerance undefined
                           0.0,                             // Tolerance value (not used)
                           0.0,                             // Start time
                           fmi2False,                       // Stop time undefined 
                           1.0 );                           // Stop time (not used)
                           
      // FMI2 component initialization
      fmi2EnterInitializationMode( (fmi2Component) block->work );
      fmi2ExitInitializationMode( (fmi2Component) block->work );
      
      // Event iteration
      eventInfo.newDiscreteStatesNeeded = fmi2True;
      while( eventInfo.newDiscreteStatesNeeded )
      {
        // update discrete states
        fmi2NewDiscreteStates( (fmi2Component) block->work, &eventInfo );
        // if( eventInfo.terminateSimulation ) goto TERMINATE_MODEL;
      }
      
      fmi2EnterContinuousTimeMode( (fmi2Component) block->work );
      
      // Retrieve initial state x
      fmi2GetContinuousStates( (fmi2Component) block->work, block->x, block->nx );
      
      // Retrieve solution at t=Tstart, for example for outputs
      double** y = (double**) block->outptr;
      double outputs[ block->nout ];
      const fmi2ValueReference* outputIndexes = // Have to figure out how to obtain that
      fmi2GetReal( (fmi2Component) block->work, outputIndexes, outputs, block->nout );
      for( int i = 0; i < block->nout; i++ )
        y[ i ][ 0 ] = outputs[ i ];

      // For now, we assume no discrete states
        
      break;
    }

    /* ... */
  }

  return;
}
{% endhighlight %}

(Error handling is not shown to simplify visualization)

### [De/Re]initialization

As we are describing initialization procedures for **FMI2**-based blocks, it makes sense to also talk about ending and reinitializing them:

<table style="width:100%">
  <tr> <th>flags</th> <th>                inputs                </th> <th>  outputs   </th> <th>               description                     </th> </tr>
  <tr> <td> ... </td> <td>                  ...                 </td> <td>    ...     </td> <td>                  ...                          </td> </tr>
  <tr> <td>  5  </td> <td>              x, z, inptr             </td> <td>x, z, outptr</td> <td> final call to block for ending the simulation </td> </tr>
  <tr> <td>  6  </td> <td> t, nevprt, x, z, inptr, mode, phase  </td> <td>x, z, outptr</td> <td>       reinitialization (if needed)            </td> </tr>
  <tr> <td> ... </td> <td>                  ...                 </td> <td>    ...     </td> <td>                  ...                          </td> </tr>
</table>

From the **documentation**:

*"Ending: This job is called when flag=5 at the end. This case is used to close files opened by the block at the begining or during the simulation, to free the allocated memory, etc."*

*"Reinitialization: This job is called under flag=6. In this case, the values of the inputs are available and the function create a fixed point iteration for the outputs of the block. On this occasion, the function can also reinitialize its initial states."*

{% highlight cpp %}
/* ... */

void fmi2_block( scicos_block* block, const int flag )
{
  /* ... */

  switch( flag )
  {
    /* ... */
        
    case Ending:                    // Value 5, part of a enumeration defined in scicos_block4.h
    {
      fmi2Terminate( (fmi2Component) block->work );       // Terminate simulation for this component
            
      // Retrieve solution at t=Tend, for example for outputs
      double** y = (double**) block->outptr;
      double outputs[ block->nout ];
      const fmi2ValueReference* outputIndexes = // Have to figure out how to obtain that
      fmi2GetReal( (fmi2Component) block->work, outputIndexes, outputs, block->nout );
      for( int i = 0; i < block->nout; i++ )
        y[ i ][ 0 ] = outputs[ i ];
            
      fmi2FreeInstance( (fmi2Component) block->work );    // Deallocate memory
        
      break;
    }
    
    case ReInitialization:          // Value 6, part of a enumeration defined in scicos_block4.h
    {
      fmi2Reset( (fmi2Component) block->work );       // Reset simulation to initial values
      
      // But still in sync with simulator time
      fmi2SetTime( (fmi2Component) block->work, get_scicos_time() );
        
      // Retrieve initial state x
      fmi2GetContinuousStates( (fmi2Component) block->work, block->x, block->nx );
        
      break;
    }

    /* ... */
  }

  return;
}
{% endhighlight %}


Be aware that this implementation is subject to change, as I better understand of how to deal with the **API**. But, for now, that's all I have to show you.


Thanks for sticking with me one more time. See Ya !
