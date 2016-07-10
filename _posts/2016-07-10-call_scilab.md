---
layout: post
title: My 2016 GSoC Project - Part X
subtitle: Better Call Scilab (but not in any way)
category: Programming
tags: [GSoC, Scilab, C++]
---   

Hello again, folks! It's been quite a while, hasn't it ?

As it turns out, contrary to planned, I had to spend more time on the **Jupyter** messaging classes, because my **API** sucked and didn't cover my own use case properly. Now it sucks a little less, hopefully, and I updated [my previous post about it]({% post_url 2016-06-26-have_class %}) accordingly.

On that same post, I also mentioned that I have a really crucial task for this second half of my **GSoC** project. That's how I visualized **Clément David** telling me it:

<p align="center">
  <img src="https://cdn.meme.am/instances/500x/69257736.jpg">
  (Poser alert: I've never actually watched the series)
</p>

Just kidding, Clément. But it's true that I started working on that.

This post was meant to have been posted a week ago, as I thought there was some preliminary results to be shown. However, it looks like I got the **Scilab** developers idea for this project a little wrong.

My intention was to create the **Jupyter** kernel as an independent application that would call **Scilab** libraries through its external **API**, capture the text output and send it to the **frontend** in question. The thing I didn't know is that this interface is quite limited (you basically offload work to the library and forget it), and it would be difficult to use it for providing all the information a **Jupyter** client could request (like **variable inspection** and **command history**).

That's why all along the developers' plan seemed to be using **Scilab's internal API**, rewritten in **C++** for version **6.0**, with which it would be possible to obtain all the needed data more easily.

The thing is: the way I understand it, that means the **Jupyter** server will be a part of **Scilab**'s core itself, more precisely a new interface, which makes sense, as **Scilab** already could be launched with different interfaces (the **command line/terminal** and the **Java-based graphical** one).

As I got some progress with the wrong (for this case) approach, and wrapping my head around the new one is taking some time, I decided to write about it anyways. For anyone who is planning to just follow my progress, this post could pretty much be ignored, but I think it could still be an interesting read for someone (I hope).

### Call_scilab and api_scilab

Since version **5.2**, **Scilab** defines the [**call_scilab API**](https://help.scilab.org/docs/6.0.0/en_US/call_scilab.html) for sending string commands (or **jobs**) to its internal engine from **C/C++** applications. That's like the expected behaviour we want when our **Jupyter** kernel receives an **execute_request** message. 

Two functions are available for this task:

{% highlight cpp %}
// Send a single string command to Scilab engine
int SendScilabJob( char *job );

// Send a list of string commands to Scilab engine
int SendScilabJobs( char **jobs, int numberjobs );
{% endhighlight %}

There is also the more complex [**api_scilab API**](https://help.scilab.org/docs/6.0.0/en_US/api_scilab.html), for manipulating its native objects (e.g. matrices) directly, more geared towards developing **C/C++** extension modules for the offcial **Scilab** application, but that could (I guess) also be used in our case, with some limitations, for tasks like code instrospection when handling an **inspect_request**.

### Setting it up

By linking **Scilab** libraries and its [**Java Virtual Machine**](https://en.wikipedia.org/wiki/Java_virtual_machine) (**JVM**) dependencies into a **C/C++** program, a **Scilab** engine can be integrated for data manipulation and commands processing.

This is basically the way to perform it (adapted from [**Scilab Help Pages**](https://help.scilab.org/docs/6.0.0/en_US/call_scilab.html)):

{% highlight cpp %}
// A simple call_scilab example

#include <stdio.h> /* stderr */

#include "api_scilab.h" /* Provide functions to access to the memory of Scilab */
#include "call_scilab.h" /* Provide functions to call Scilab engine */

int main( void )
{
 /****** INITIALIZATION **********/
 #ifdef _MSC_VER
 if ( StartScilab( NULL, NULL, 0 ) == FALSE )
 #else
 if ( StartScilab( getenv( "SCI" ), NULL, 0 ) == FALSE )
 #endif
 {
   fprintf( stderr,"Error while calling StartScilab\n" );
   return -1;
 }

 /****** ACTUAL Scilab TASKS *******/

 // Data manipulation or SendScilabJob* calls 

 /****** TERMINATION **********/
 if( TerminateScilab( NULL ) == FALSE ) 
 {
   fprintf( stderr,"Error while calling TerminateScilab\n" );
   return -2;
 }
 
 return 0;
}
{% endhighlight %}

You can see that, on non **Windows** systems (namely **Unix** ones), the created engine need to look for the **SCI** environment variable, which should be previously initialized and contain the path to **Scilab** base directory.

For our **Jupyter** kernel, I'm using a **bash script** to set this value before calling the actual executable:

- ScilabKernelLauncher.sh

{% highlight bash %}
#!/bin/bash

export SCI_PATH=/path/to/scilab/installation/       # normally /usr
export SCI=${SCI_PATH}/share/scilab
export JAVA_HOME=/path/to/java/installation/        # normally /usr/lib/jvm/java-8-openjdk/
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$SCI_PATH/lib/scilab/:$JAVA_HOME/jre/lib/amd64/:$JAVA_HOME/jre/lib/amd64/server/:$JAVA_HOME/jre/lib/amd64/native_threads/

# gets name of the directory where the script is
KERNEL_DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"

# calls the actual kernel passing the command line arguments
${KERNEL_DIR}/ScilabKernel $@
{% endhighlight %}

As we are now calling the compiled kernel through a script, the kernel spec file should be changed accordingly:

- kernel.json

{% highlight python %}
{
 "argv": [ "$HOME/.local/share/jupyter/kernels/Scilab/ScilabKernelLauncher.sh", "{connection_file}" ],
 "display_name": "Scilab Native",
 "language": "scilab"
} 
{% endhighlight %}

### Output redirection

As I said in the beggining of the post, the **call_scilab API** is kind of a [black box](https://en.wikipedia.org/wiki/Black_box): you pass the command strings and just visualize the output, as text or graphs, like a user of the main **Scilab** application. No function is provided to get the processing results as a nicely formatted string.

Some years ago, I followed the **GSoC** project of another brazilian, **Felipe Saraiva**, who worked on a [**Scilab** backend for **Cantor**](https://wiki.scilab.org/Contributor%20-%20Scilab%20backend%20to%20Cantor), a scientific application from **KDE project** that works more or less like **Jupyter's QtConsole**, providing a common (non remote) interface for different scientific languages. Reading from his reports, I found out that he was initially also planning to use **call_scilab** and somehow read from the standard output streams (**stdout** and **stderr**, in **C** programs) to get the result from user commands. So that's what I've been trying to do.

After some battling with [**pipe related system calls**](https://en.wikipedia.org/wiki/Dup_(system_call)), I got it partially working, and was able to capture text output, when the **SendScilabJob** call writes something to **stdout** (the terminal in a console application):

<p align="center">
  <img src="/img/qtconsole.png">
  (As you can see, it only works when I ask to display de value explicitly)
</p>

### Wrapping it up

I tried to write this new functionality in a more object-oriented way from the beggining, to avoid the need to refactor later. I ended up with a simple, mostly static, class, that I show here entirely, as I'll need to use other **API** and this code probably won't be part of the final kernel implementation:

{% highlight cpp %}
#ifndef KERNEL_BACKEND_HPP
#define KERNEL_BACKEND_HPP

#ifdef _MSC_VER
  #include <io.h>
  #define popen _popen 
  #define pclose _pclose
  #define stat _stat 
  #define dup _dup
  #define dup2 _dup2
  #define fileno _fileno
  #define close _close
  #define pipe _pipe
  #define read _read
  #define eof _eof
#else
  #include <unistd.h>
#endif

#include <fcntl.h>
#include <stdio.h>

#include <iostream>
#include <string>

#include "api_scilab.h" /* Provide functions to access to the memory of Scilab */
#include "call_scilab.h" /* Provide functions to call Scilab engine */

class KernelBackend
{
public:
  static int Initialize()
  {
    if( !initialized )
    {
      // Make stdout unbuffered so there is no need to call fflush() on it
      //setvbuf( stdout, NULL, _IONBF, 0 );
      
      #ifdef _MSC_VER
      if( pipe( outputPipePair, 65536, O_BINARY ) == -1 )
      #else
      if( pipe( outputPipePair ) == -1 )
      #endif
      {
        perror( "Error creating output pipes" );
        return -1;
      }
      
      #ifdef _MSC_VER
      if( StartScilab( NULL, NULL, 0 ) == FALSE )
      #else
      if( StartScilab( getenv( "SCI" ), NULL, 0 ) == FALSE )
      #endif
      {
        std::cerr << "Error while calling StartScilab" << std::endl;
        return -1;
      }
      
      initialized = true;
    }
    
    return 0;
  }
  
  static int Terminate()
  {
    if( initialized )
    {
      if( TerminateScilab( NULL ) == FALSE ) 
      {
        std::cerr << "Error while calling TerminateScilab" << std::endl;
        return -1;
      }
      
      if( outputPipePair[ READ ] > 0 ) close( outputPipePair[ READ ] );
      if( outputPipePair[ WRITE ] > 0 ) close( outputPipePair[ WRITE ] );
    }
    
    return 0;
  }
  
  static std::string ProcessCommand( std::string code )
  {
    char outputBuffer[ BUFSIZ ] = "";
    std::string outputString;
    
    if( initialized and (not code.empty()) )
    {  
      int stdoutFDBackup = dup( STDOUT_FILENO );
      if( stdoutFDBackup == -1 )
        perror( "Error copying stdout file descriptor" );
      
      if( dup2( outputPipePair[ WRITE ], STDOUT_FILENO ) == -1 )
        perror( "Error redirecting stdout to output pipe" );
      
      SendScilabJob( (char*) code.data() );
      printf( "\n" );
    
      fflush( stdout );
      if( read( outputPipePair[ READ ], outputBuffer, BUFSIZ ) > 0 )
        outputString += outputBuffer;
      
      if( dup2( stdoutFDBackup, STDOUT_FILENO ) == -1 )
        perror( "Error restoring stdout" );
      
      if( stdoutFDBackup > 0 ) close( stdoutFDBackup );
    }
    
    return outputString;
  }
  
private:
  static bool initialized;
  
  enum { READ, WRITE };
  static int outputPipePair[ 2 ];
};

bool KernelBackend::initialized = false;

int KernelBackend::outputPipePair[ 2 ];

#endif // KERNEL_BACKEND_HPP
{% endhighlight %}

If on one side I lost time taking the wrong path without previously consulting my mentors (shame on me, lol), on the other hand it was a good learning experience, since I knew very little about **pipes** and the possibilities for their usage in an application.

I hope to catch up again soon. See you next time !!
