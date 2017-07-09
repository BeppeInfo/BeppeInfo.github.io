---
layout: post
title: My 2017 GSoC Project - Part IV
subtitle: (Un)Filling The Box
category: Programming
tags: [GSoC-2017, Scilab, Modelica]
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

- *"Initialization: At the begining of the simulation, this job is called under flag=4 to initialize continuous and discrete states (if necessary). It also initialize the output port of the block. This function is not used in all of the blocks, it is used for blocks that needed dynamically allocated memory (the allocation is done under this flag), for blocks that read and write data from files, for opening a file, or by the scope to initialize the graphics window."*

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
  int nin;        // Number of regular input ports
  int* insz;      // Vector of sizes of regular input ports
  void** inptr;   // Tables of pointer to the regular input ports
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

#include "fmi2TypesPlatform.h"
#include "fmi2FunctionTypes.h"
#include "fmi2Functions.h"

// Enumeration of block->work contents
enum { MODEL, STATE_REFS, STATE_DERIV, STATE_DERIV_REFS, INPUT, INPUT_REFS, OUTPUT, OUTPUT_REFS, CONTENTS_NUMBER };

// Simple placeholder logging message
static void print_log_message( fmi2ComponentEnvironment componentEnvironment, fmi2String instanceName, 
                        fmi2Status status, fmi2String category, fmi2String message, ... )
{
  va_list args;
  va_start( args, message );
  vprintf( (const char*) message, args );
  va_end( args );
}

// Utility function for setting Scicos block -> FMI2 model input
static void set_input(scicos_block* block)
{
    // Considering input only floating point continuous values
    double** u = (double**) block->inptr;
    fmi2Real* input = (fmi2Real*) block->work[ INPUT ];
    fmi2ValueReference* input_refs = (fmi2ValueReference*) block->work[ INPUT_REFS ];
    // Set u inputs before calculation derivatives
    for( int i = 0; i < block->nin; i++ )
        input[ i ] = u[ i ][ 0 ];
    fmi2SetReal( (fmi2Component) block->work[ MODEL ], input_refs, block->nin, input );
}

// Utility function for getting FMI2 model -> Scicos block output
static void get_output(scicos_block* block)
{
    // Considering output only floating point continuous values
    double** y = (double**) block->outptr;
    fmi2Real* output = (fmi2Real*) block->work[ INPUT ];
    fmi2ValueReference* output_refs = (fmi2ValueReference*) block->work[ INPUT_REFS ];
    // Retrieve output values
    if( fmi2GetReal( (fmi2Component) block->work[ MODEL ], output_refs, block->nout, output ) == fmi2OK )
    {
        // Setting outputs the same as continuous states
        for( int i = 0; i < block->nout; i++ )
            y[ i ][ 0 ] = output[ i ];
    }
    
    // For now, we assume no discrete outputs
}

// The computational function itself
void fmi2_block( scicos_block* block, const int flag )
{
    static fmi2EventInfo eventInfo;

    switch( flag )
    {
        /* ... */
        
        // Flag 4: State initialisation and memory allocation
        case Initialization:                    
        {
            // User provided functions for FMI2 data management
            const fmi2CallbackFunctions functions = { .logger = print_log_message,
                                                      .allocateMemory = calloc,
                                                      .freeMemory = free, 
                                                      .stepFinished = NULL,
                                                      .componentEnvironment = NULL };
            
            block->work = (void**) calloc( CONTENTS_NUMBER, sizeof(void*) );
                                                      
            // Store FMI2 component in the work field     
            block->work[ MODEL ] = fmi2Instantiate( block->label,          // Instance name 
                                                    fmi2ModelExchange,     // Exported model type
                                                    block->uid,            // Model GUID, have to be obtained
                                                    "file:/data_path/",    // Optional FMU resources location
                                                    &functions,            // User provided callbacks
                                                    fmi2True,              // Interactive mode
                                                    fmi2True );            // Logging On
            
            // Create state variables references list
            block->work[ STATE_REFS ] = calloc( block->nx, sizeof(fmi2ValueReference) );
            block->work[ STATE_DERIV ] = calloc( block->nx, sizeof(fmi2Real) );
            block->work[ STATE_DERIV_REFS ] = calloc( block->nx, sizeof(fmi2ValueReference) );
            for( int i = 0; i < block->nx; i++ )
            {
                ((fmi2ValueReference*) block->work[ STATE_REFS ])[ i ] = (fmi2ValueReference) i;
                ((fmi2ValueReference*) block->work[ STATE_DERIV_REFS ])[ i ] = (fmi2ValueReference) ( block->nx + i );
            }
            
            // Create input variables references list
            block->work[ INPUT ] = calloc( block->nin, sizeof(fmi2Real) );
            block->work[ INPUT_REFS ] = calloc( block->nin, sizeof(fmi2ValueReference) );
            
            // Need to set input references based on model description (to be implemented)

            set_input( block );           // Set initial inputs
            
            // Define simulation parameters. Internally calls state and event setting functions, 
            // which should be called before any model evaluation/event processing ones
            fmi2SetupExperiment( (fmi2Component) block->work[ MODEL ],     // FMI2 component   
                                fmi2False,                                // Tolerance undefined
                                0.0,                                      // Tolerance value (not used)
                                0.0,                                      // Start time
                                fmi2False,                                // Stop time undefined 
                                1.0 );                                    // Stop time (not used)
              
            // FMI2 component initialization
            fmi2EnterInitializationMode( (fmi2Component) block->work[ MODEL ] );
            fmi2ExitInitializationMode( (fmi2Component) block->work[ MODEL ] );
              
            // Event iteration
            eventInfo.newDiscreteStatesNeeded = fmi2True;
            while( eventInfo.newDiscreteStatesNeeded )
            {
                // update discrete states
                fmi2NewDiscreteStates( (fmi2Component) block->work[ MODEL ], &eventInfo );
                // if( eventInfo.terminateSimulation ) goto TERMINATE_MODEL;
            }
              
            // Enable iterative derivatives calculation and integration
            fmi2EnterContinuousTimeMode( (fmi2Component) block->work[ MODEL ] );
              
            fmi2GetContinuousStates( (fmi2Component) block->work[ MODEL ], block->x, block->nx );
            
            // Create output variables references list
            block->work[ OUTPUT ] = calloc( block->nout, sizeof(fmi2Real) );
            block->work[ OUTPUT_REFS ] = calloc( block->nout, sizeof(fmi2ValueReference) );
            
            // Need to set output references based on model description (to be implemented)
            
            get_output( block );          // Get initial outputs
              
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

- *"Ending: This job is called when flag=5 at the end. This case is used to close files opened by the block at the begining or during the simulation, to free the allocated memory, etc."*

- *"Reinitialization: This job is called under flag=6. In this case, the values of the inputs are available and the function create a fixed point iteration for the outputs of the block. On this occasion, the function can also reinitialize its initial states."*

{% highlight cpp %}
/* ... */

void fmi2_block( scicos_block* block, const int flag )
{
    /* ... */

    switch( flag )
    {
        /* ... */
        
        // Flag 5: simulation end and memory deallication
        case Ending:                    
        {
            // Get final state and output
        
            fmi2SetTime( (fmi2Component) block->work[ MODEL ], get_scicos_time() );
            
            fmi2GetContinuousStates( (fmi2Component) block->work[ MODEL ], block->x, block->nx );
            
            get_output( block );
            
            fmi2Terminate( (fmi2Component) block->work[ MODEL ] );       // Terminate simulation for this component
            
            // Deallocate memory
            
            free( block->work[ STATE_REFS ] );
            free( block->work[ STATE_DERIV ] );
            free( block->work[ STATE_DERIV_REFS ] );
            free( block->work[ INPUT ] );
            free( block->work[ INPUT_REFS ] );
            
            fmi2FreeInstance( (fmi2Component) block->work[ MODEL ] );
            
            break;
        }
        // Flag 6: Output state initialisation
        case ReInitialization:
        {
            fmi2Reset( (fmi2Component) block->work[ MODEL ] );       // Reset simulation to initial values
            
            // Get reinitialized model state and output
            
            fmi2SetTime( (fmi2Component) block->work[ MODEL ], get_scicos_time() );
            
            fmi2GetContinuousStates( (fmi2Component) block->work[ MODEL ], block->x, block->nx );
            
            get_output( block );
            
            break;
        }

        /* ... */
    }

    return;
}
{% endhighlight %}


Be aware that this implementation is subject to change, as I better understand of how to deal with the **API**. But, for now, that's all I have to show you.


Thanks for sticking with me one more time. See Ya !
