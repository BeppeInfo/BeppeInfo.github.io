---
layout: post
title: My 2016 GSoC Project - Part XV
subtitle: Learning to Read
category: Programming
tags: [GSoC, Scilab, C++, Jupyter]
---     

We meet again, companions.

This latest (and most likely last) **GSoC** achievement I'm going to talk about was probably one of the most difficult ones, whose hardness was potentialized by the urgency of the **Final Term** week. Its stressful to try finding the right screws to twist when there's a time bomb by your side. *Lol*

Enough of metaphors. Let's go to what matters (maybe...).

### User Input

As can be implied from the previous post, input management was one of the last big missing pieces of our **Jupyter Scilab** kernel and, as experienced showed, a rather troublesome one. Being the only type of message exchange (**request** and **reply**) for what the request is dispatched from kernel, and even during the processing of a previously received command, its implement breaks the **wait for request -> process -> send reply** control flow entirely.

- **input_request** message specification
{% highlight python %}
content = {
    # the text to show at the prompt
    'prompt' : str,
    # Is the request for a password?
    # If so, the frontend shouldn't echo input.
    'password' : bool
}
{% endhighlight %}

- **input_reply** message specification
{% highlight python %}
content = { 'value' : str }
{% endhighlight %}

On the bright side, this breakage would be needed anyways, for allowing the kernel to process high priority requests (mainly **shudown**) during long command executions. That was solved by spamming a detached thread to wait for execution results (that can issue output or input requests), and immediately returning on the main update thread to poll for new messages:

- Our **HandleExecutionRequest** method, from **JupyterKernel** class
{% highlight cpp %}
void JupyterKernel::HandleExecutionRequest( JupyterKernelConnection& publisher, 
                                            JupyterMessage& commandMessage )
{
  static unsigned int executionCount;
  
  JupyterMessage statusMessage = commandMessage.GenerateReply( USER_NAME, "status" );
  statusMessage.content[ "execution_state" ] = "busy";
  publisher.SendMessage( statusMessage );
  
  Json::Value& commandContent = commandMessage.content;
  bool silent = commandContent.get( "silent", false ).asBool();
  bool storeHistory = commandContent.get( "store_history", not silent ).asBool();
  std::string commandString = commandContent.get( "code", "" ).asString();
  
  // Some clients doesn't allow input requests and can inform it
  isInputAllowed = commandContent.get( "allow_stdin", true ).asBool();
  
  if( not silent ) // Results are not published if client requests silent execution
  {
    JupyterMessage inputMessage = commandMessage.GenerateReply( USER_NAME, "execute_input" );
    inputMessage.content[ "execution_count" ] = executionCount;
    inputMessage.content[ "code" ] = commandString;
    publisher.SendMessage( inputMessage );
  }
  
  if( storeHistory ) // History is only updated if required
  {
    appendLineToScilabHistory( (char*) commandString.data() );
    executionCount++;
  }
  
  // We don't need to store a handle for this thread, as a 
  // detached thread doesn't need to be awaited (joined) for returning
  std::thread( &HandleExecutionReply, std::ref( publisher ), commandMessage ).detach();
  
  // Set the new command input string and unlock its reading by the internal engine
  inputCurrentString = commandString;
  inputLock.unlock();
  isProcessing = true;
}
{% endhighlight %}

With execution thread free to run independently, I turn my attention to two big concerns I had in this project since the beginning: how to detect when execution ends and when the console queries user for input during it.

For the second, I soon noticed the existence of a **ConsoleIsWaitingForInput** function on the scilab module folder. So... Nice and easy, right ?

- From **ConsoleIsWaitingForInput.cpp**
{% highlight cpp %}
#include "ConsoleIsWaitingForInput.hxx"
/*--------------------------------------------------------------------------*/
#include "CallScilabBridge.hxx"
using namespace org_scilab_modules_gui_bridge;
BOOL ConsoleIsWaitingForInput(void)
{
    if (getScilabJavaVM())
    {
        return booltoBOOL(CallScilabBridge::isWaitingForInput(getScilabJavaVM()));
    }
    return FALSE;
}
{% endhighlight %}

Damn. Not quite... The function is only useful when the **Java Virtual Machine** is running, which is not the case when we're not using **GUI** functionality. But there should be another way to do it, as **Scilab** actually has a **CLI** shell.

*So... What to do ?*

When the interfaces for requesting or setting values doesn't seem that obvious, I usually I see no other option but to dig deep into codepaths of functions that supposedly do the same thing that I want my code to do. In this case, one that quickly comes to mind is **Scilab**'s **input** function:

- From **input.sci**
{% highlight octave %}
function [x] = input(msg, flag)
// [...]

        prompt("");
        mprintf(msg);
        x = mscanf(fmt);
        
// [...]
{% endhighlight %}

You don't need to be that used to **Scilab** language to think this is the relevant part for us. The name **mscanf** reminds of **C**'s **scanf** function for getting console input, so we should be on the right track. The **mscanf** is another **Scilab** function, implemented in **C++**:

- From **sci_mscanf.cpp**
{% highlight cpp %}
function [x] = input(msg, flag)
/* [...] */
        // get data
        // The console thread must not parse the next console input.
        ConfigVariable::setScilabCommand(0);

        // Get the console input filled by the console thread.
        char* pcConsoleReadStr = ConfigVariable::getConsoleReadStr();
        ThreadManagement::SendConsoleExecDoneSignal();
        while (pcConsoleReadStr == NULL)
        {
            pcConsoleReadStr = ConfigVariable::getConsoleReadStr();
            ThreadManagement::SendConsoleExecDoneSignal();
        }

        // reset flag to default value
        ConfigVariable::setScilabCommand(1);
        
/* [...] */
{% endhighlight %}

Bingo ! As **setScilabCommand** method is not used anywhere else, we can assume that anytime the internal variable is set to **0**, which can be acessed through the correspondent **isScilabCommand** method, the console is waiting for user input.

### Execution Management

Now, the first problem, of verifying command execution completion (or console readiness for receiving a new one), although sounding simple, was a rather annoying one. In summary, after trying to play with the **ThreadManagement** class methods (based on the code I was studying), and getting a lot of [**deadlocks**](https://en.wikipedia.org/wiki/Deadlock) in the process, I followed **Clément**'s suggestion to look at **sci_execstr.cpp** source file:

- From **sci_execstr.cpp**
{% highlight cpp %}
/* [...] */

    //save current prompt mode
    int iPromptMode = ConfigVariable::getPromptMode();
    ConfigVariable::setPromptMode(-1);

    /* [...] */

    ast::SeqExp* pSeqExp = pExp->getAs<ast::SeqExp>();
    std::unique_ptr<ast::ConstVisitor> run(ConfigVariable::getDefaultVisitor());
    try
    {
        symbol::Context* pCtx = symbol::Context::getInstance();
        int scope = pCtx->getScopeLevel();
        int level = ConfigVariable::getRecursionLevel();
        try
        {
            pSeqExp->accept(*run);
        }
        /* [...] */
    }
    catch (const ast::InternalError& ie)
    {
        /* [...] */
    }

    /* [...] */

    ConfigVariable::macroFirstLine_end();
    ConfigVariable::setPromptMode(iPromptMode);
    
/* [...] */
{% endhighlight %}

Something that I also noticed on other functions that handle code execution, but which I was neglecting so far, is the manipulation of the **prompt** mode. Some brief inspection of the **ConfigVariable** class header file showed that the value **-1** represents the **silent** mode, defined before execution, while **2** indicates the avaibility for new commands (**prompt** mode).

I don't think that considering only these two modes is enough, but they cover most of the cases, and allowed me to ditch **ThreadManagement** signals and waits to have a somehow working execution management:

- Our **HandleExecutionReply** method (runs in a detached thread), from **JupyterKernel** class
{% highlight cpp %}
void JupyterKernel::HandleExecutionReply( JupyterKernelConnection& publisher, 
                                          JupyterMessage commandMessage )
{
  static unsigned int executionCount;
  
  Json::Value& commandContent = commandMessage.content;
  bool silent = commandContent.get( "silent", false ).asBool();
  bool storeHistory = commandContent.get( "store_history", not silent ).asBool();
  std::string shellIdentity = commandMessage.identifier;
  
  JupyterMessage resultMessage = commandMessage.GenerateReply( USER_NAME, "execute_result" );
  
  while( isProcessing )
  {
    // Kinda hacky. No synchronization. Give time for the core Scilab threads to process
    std::this_thread::sleep_for( std::chrono::milliseconds( 100 ) );
    
    // We're waiting form input. Request it from the client
    if( not ConfigVariable::isScilabCommand() ) HandleInputRequest( publisher, shellIdentity );
    // Run until the command prompt is available again
    else if( ConfigVariable::getPromptMode() == 2 ) break;
  }
  
  // Helper function. Concatenates the output queue in a single string
  std::string resultString = GetCurrentOutput();
  
  if( not silent ) 
  {
    resultMessage.content[ "execution_count" ] = executionCount;
    resultMessage.content[ "data" ] = Json::Value( Json::objectValue );
    resultMessage.content[ "data" ][ "text/plain" ] = resultString;
    resultMessage.content[ "metadata" ] = Json::Value( Json::objectValue );
    publisher.SendMessage( resultMessage );
  }
  
  
  JupyterMessage replyMessage = commandMessage.GenerateReply( USER_NAME );
  
  // Basic reply and error management
  /* [...] */
  
  publisher.SendMessage( replyMessage );
  
  if( storeHistory ) 
  {
    // Someday...
    //appendLineToScilabHistory( (char*) resultString.data() );
    executionCount++;
  }
  
  JupyterMessage statusMessage = commandMessage.GenerateReply( USER_NAME, "status" );
  statusMessage.content[ "execution_state" ] = "idle";
  publisher.SendMessage( statusMessage );
  
  isProcessing = false;
}
{% endhighlight %}

- Our **HandleInputRequest** method, from **JupyterKernel** class
{% highlight cpp %}
void JupyterKernel::HandleInputRequest(JupyterKernelConnection& requester, std::string shellIdentity)
{
  // Prevents it from being called multiple times for the same mscanf
  if( isWaitingInput ) return;
  
  if( not isInputAllowed )
  {
    ConfigVariable::setConsoleReadStr( (char*) "" );
    return;
  }
  
  isWaitingInput = true;
  
  // Use all the previous results as the prompt string
  std::string promptString = GetCurrentOutput();
    
  JupyterMessage inputRequestMessage( USER_NAME, "input_request" );
  
  // Input requests have to use the same identity as the client shell
  inputRequestMessage.identifier = shellIdentity;
    
  inputRequestMessage.content[ "prompt" ] = promptString;
  inputRequestMessage.content[ "password" ] = false;
    
  requester.SendMessage( inputRequestMessage );
}
{% endhighlight %}

- Our **HandleInputReply** method, from **JupyterKernel** class
{% highlight cpp %}
void JupyterKernel::HandleInputReply( Json::Value& replyContent )
{
  if( not isWaitingInput ) return;
  
  std::string stdinString = replyContent.get( "value", "" ).asString();

  isWaitingInput = false;

  inputCurrentString = stdinString;
  inputLock.unlock();
}
{% endhighlight %}


### Results

With all set up, the bug of a new **Jupyter** prompt appearing appearing before the results of the previous command is fixed (actually, it is much less likely to happen...), and input commands work as intended. To be fair, there are still corner cases I can think of and synchronization issues that I wish to fix for sure (hopefully, without abusing [**mutexes**](https://en.wikipedia.org/wiki/Mutual_exclusion)), but that'll have to come later:

<p align="center">
  <img src="/img/qtconsole_input.png">
</p>

For now, I feel more or less satisfied... *Ciao*
