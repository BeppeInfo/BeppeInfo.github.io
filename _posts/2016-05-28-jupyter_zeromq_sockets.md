---
layout: post
title: My 2016 GSoC Project - Part IV
subtitle: Jupyter ZeroMQ Sockets
category: Programming
tags: [GSoC, ZeroMQ, Jupyter]
--- 

Information found at the **Jupyter** documentation pages left me a little confused. My first impression was that communication between frontend and kernel is performed through 3 **ZeroMQ** connections: Shell (Router-Dealer), IOPub (Publisher-Subscriber) and Stdin (Router-Dealer). Further reading of the docs mentioned other 2 connection channels, Control and Heartbeat, but there was less info about their roles and how to implement them.

Fortunately, [a blog post by Andrew Gibiansky](http://andrew.gibiansky.com/blog/ipython/ipython-kernels/), the creator of the [Haskell](https://www.haskell.org/) language kernel for **IPython**, clarified things.

Obviously some informations are outdated, as **IPython** is the predecessor of **Jupyter**, but the general architecture remained the same:

<p align="center">
  <img src="http://andrew.gibiansky.com/blog/ipython/ipython-kernels/images/ipython.png">
</p>

According to the article, the **ZeroMQ** socket types for each of those connections are the following:

- Heartbeat: Reply (Request on frontends)
- Shell, Control, Stdin: Router (Dealer on frontends)
- IOPub: Publisher (Subscriber on frontends)

Now that we got the general idea, let's move to the implementation.

It's pretty trivial to write the code to just instantiate the **ZeroMQ** connections, but there was some details I didn't know.

As mentioned in the previous post, another **IPython** project, [the simple_kernel.py by Doug Blank](https://github.com/dsblank/simple_kernel), gave me some insights about how to make things work: creating a separate thread for the **Heartbeat** socket guarantees that it answers immediately to the client requests (some kind of **"Are you there yet ?"** messages), keeping it aware that the backend is still running. 

Thanks to the [**C++11** standard threads](http://www.cplusplus.com/reference/thread/thread/), thread creation and management in **C++** can be done in a portable manner.

The kernel code that I have for now is shown below (and you can also see it on [my GitHub repository](https://github.com/Bitiquinho/Jupyter-Scilab-Kernel)). Obviously it is not functional, as this only create the required sockets and runs the **Heartbeat** thread:

{% highlight c %}
#include <zmq.hpp>      // ZeroMQ functions
#include <string>
#include <iostream>
#include <thread>       // C++11 threads
#include <chrono>       // C++11 time handling functions
#include <signal.h>     // Signal handling functions
#ifndef _WIN32
  #include <unistd.h>
#else
  #include <windows.h>
#endif

// This should be "volatile" because of low-level 
// CPU registers stuff. Let it be for now.
static volatile bool isRunning = true;
void HandleExitSignal( int dummy )
{
  isRunning = false;    // Self-explanatory enough
}

// Heartbeat thread function declaration
void HeartbeatLoopRun( zmq::context_t* );

int main( int argc, char* argv[] ) 
{
  if( argc > 1 )
  {
    std::cout << "Reading connection file " << argv[ 1 ] << std::endl;
  }

  // Set interrupt function to be called when Ctrl^C is pressed
  signal( SIGINT, HandleExitSignal );

  // Every ZeroMQ application should create its own unique context
  zmq::context_t context( 1 );

  // Creating I/O Pub socket on arbitrary port
  zmq::socket_t ioPubSocket( context, ZMQ_PUB );
  ioPubSocket.bind( "tcp://*:50002" );

  // Creating Control socket on arbitrary port
  zmq::socket_t controlSocket( context, ZMQ_ROUTER );
  controlSocket.bind( "tcp://*:50003" );

  // Creating Stdin socket on arbitrary port
  zmq::socket_t inputSocket( context, ZMQ_ROUTER );
  inputSocket.bind( "tcp://*:50004" );

  // Creating Shell socket on arbitrary port
  zmq::socket_t shellSocket( context, ZMQ_ROUTER );
  shellSocket.bind( "tcp://*:50005" );

  // Run Heartbeat thread. Context is thread-safe, so we can safely pass it
  std::thread heartbeatThread( HeartbeatLoopRun, &context );

  while( isRunning ) // Run while we do not press Ctrl^C
  {
    // Let the Heartbeat thread do its job ( What a call !! )
    std::this_thread::sleep_for( std::chrono::milliseconds( 1 ) );
  }

  heartbeatThread.join(); // Wait for the Heartbeat thread to return

  return 0;
}

// Heartbeat thread function definition
void HeartbeatLoopRun( zmq::context_t* ptr_context )
{
  // Creating Heartbeat socket on arbitrary port on its own thread
  zmq::socket_t heartbeatSocket( *ptr_context, ZMQ_REP );
  heartbeatSocket.bind( "tcp://*:50001" );

  while( isRunning )
  {
    zmq::message_t ping; // We recreate message each time as "send" nullifies it
    
    heartbeatSocket.recv( &ping ); // Client asks if kernel is still running
    
    std::cout << "Received Heartbeat message: " << (char*) ping.data() << std::endl;
    
    heartbeatSocket.send( ping ); // Answer immediately
  }
}
{% endhighlight %}


Having compiled the code, we setup the configuration files according to the [Jupyter docs](http://jupyter-client.readthedocs.io/en/latest/kernels.html#kernel-specs) to call it from the **console** (**Jupyter** client) with:

{% highlight bash %}
$ jupyter console --debug --kernel scilab
{% endhighlight %}

The result we get is the following:

{% highlight bash %}
[ZMQTerminalIPythonApp] Searching ['/home/leonardojc', '/home/leonardojc/.jupyter', '/usr/etc/jupyter', '/usr/local/etc/jupyter', '/etc/jupyter'] for config files
[ZMQTerminalIPythonApp] Looking for jupyter_config in /etc/jupyter
[ZMQTerminalIPythonApp] Looking for jupyter_config in /usr/local/etc/jupyter
[ZMQTerminalIPythonApp] Looking for jupyter_config in /usr/etc/jupyter
[ZMQTerminalIPythonApp] Looking for jupyter_config in /home/leonardojc/.jupyter
[ZMQTerminalIPythonApp] Looking for jupyter_config in /home/leonardojc
[ZMQTerminalIPythonApp] Looking for jupyter_console_config in /etc/jupyter
[ZMQTerminalIPythonApp] Looking for jupyter_console_config in /usr/local/etc/jupyter
[ZMQTerminalIPythonApp] Looking for jupyter_console_config in /usr/etc/jupyter
[ZMQTerminalIPythonApp] Looking for jupyter_console_config in /home/leonardojc/.jupyter
[ZMQTerminalIPythonApp] Looking for jupyter_console_config in /home/leonardojc
[ZMQTerminalIPythonApp] Connection File not found: /run/user/1000/jupyter/kernel-27009.json
[ZMQTerminalIPythonApp] Found kernel scilab in /home/leonardojc/.local/share/jupyter/kernels
[ZMQTerminalIPythonApp] Native kernel (python3) available from /usr/lib/python3.5/site-packages/ipykernel/resources
[ZMQTerminalIPythonApp] Starting kernel: ['/home/leonardojc/.local/share/jupyter/kernels/Scilab/ScilabKernel', '/run/user/1000/jupyter/kernel-27009.json']
[ZMQTerminalIPythonApp] Connecting to: tcp://127.0.0.1:46095
Reading connection file /run/user/1000/jupyter/kernel-27009.json
[ZMQTerminalIPythonApp] connecting shell channel to tcp://127.0.0.1:51886
[ZMQTerminalIPythonApp] Connecting to: tcp://127.0.0.1:51886
[ZMQTerminalIPythonApp] connecting iopub channel to tcp://127.0.0.1:33016
[ZMQTerminalIPythonApp] Connecting to: tcp://127.0.0.1:33016
[ZMQTerminalIPythonApp] connecting stdin channel to tcp://127.0.0.1:36901
[ZMQTerminalIPythonApp] Connecting to: tcp://127.0.0.1:36901
[ZMQTerminalIPythonApp] connecting heartbeat channel to tcp://127.0.0.1:41229
Jupyter Console 4.1.1

[ZMQTerminalIPythonApp] Starting the jupyter console mainloop...
{% endhighlight %}

As you can see, not even input is requested, and we can't send an exit command, as I didn't implement these things yet. But the kernel was correctly found and started.

You can see from the docs and the console output, there are **.json** configuration files being read. This involves an important aspect of **Jupyter** about which I'll talk later.


That's it for now. Thanks for reading and until next time !
