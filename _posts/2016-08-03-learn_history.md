---
layout: post
title: My 2016 GSoC Project - Part XIII
subtitle: Learning History to Repeat It
category: Programming
tags: [GSoC, Scilab, C++, Jupyter]
---     

Once again, I'm back !!


[Someone said at some time](http://www.goodreads.com/quotes/tag/doomed-to-repeat-it) that not knowing history dooms us to repeat it. Well, I wouldn't necessarily call repetition "doom", as there are many situations when reproducing what has been previously done is desired.

(*I'm purposely misinterpreting things for the sake of the joke*)

When working with an **interactive shell** of any interpreted language, having access to the history of last issued commands is useful for one not having to retype them over and over. On a **Linux terminal**, for example, you can browse through them using **up** and **down** arrow keys.

For that reason (I'm just guessing wildly), **Jupyter protocol** defines **history_{request,reply}** message types, with a lot of options for a client to get this information.

From the good old [**Jupyter documentation**](http://jupyter-client.readthedocs.io/en/latest/messaging.html#history):

- **history_request** structure
{% highlight python %}
content = {

  # If True, also return output history in the resulting dict.
  'output' : bool,

  # If True, return the raw input history, else the transformed input.
  'raw' : bool,

  # So far, this can be 'range', 'tail' or 'search'.
  'hist_access_type' : str,

  # If hist_access_type is 'range', get a range of input cells. session can
  # be a positive session number, or a negative number to count back from
  # the current session.
  'session' : int,
  # start and stop are line numbers within that session.
  'start' : int,
  'stop' : int,

  # If hist_access_type is 'tail' or 'search', get the last n cells.
  'n' : int,

  # If hist_access_type is 'search', get cells matching the specified glob
  # pattern (with * and ? as wildcards).
  'pattern' : str,

  # If hist_access_type is 'search' and unique is true, do not
  # include duplicated history.  Default is false.
  'unique' : bool,

}
{% endhighlight %}

- **history_reply** structure
{% highlight python %}
content = {
  # A list of 3 tuples, either:
  # (session, line_number, input) or
  # (session, line_number, (input, output)),
  # depending on whether output was False or True, respectively.
  'history' : list,
}
{% endhighlight %}


Basically, the message fields allow one to request, from the list of recorded previous commands (and their respective outputs), a continuous range of strings or the entries that match a given pattern. Regardless of how the backend gets this information, the reply must be structured in the specified format of a list of [**tuples**](https://en.wikipedia.org/wiki/Tuple).

**Scilab** has a **HistoryManager** and a **HistorySearch** classes that partially ([AFAIK](http://pt.urbandictionary.com/define.php?term=afaik)) implements those functionalities. But before jumping to hasty conclusions without further investigation, I decided to implement what seemed achievable with the current interface, and discuss later with my mentor if it should be extended to provide all the required operations.

So, at the **JupyterKernel** class, now we have an almost complete history handling method:

- From **JupyterKernel.cpp**
{% highlight cpp %}
#include "HistoryManager.h" // Wrapper C functions for HistoryManager class methods

// Other includes

void JupyterKernel::HandleExecutionRequest( JupyterKernelConnection& publisher, JupyterMessage& commandMessage, Json::Value& replyContent )
{
  // [...]

  bool silent = commandContent.get( "silent", false ).asBool();
  bool storeHistory = commandContent.get( "store_history", not silent ).asBool();
  
  // Scilab's history manager doesn't provide a way to store output separately
  // or differentiate it from input. So we're just storing input for now
  if( storeHistory ) appendLineToScilabHistory( (char*) commandString.data() );
  
  // [...]
}

void JupyterKernel::HandleHistoryRequest( Json::Value& commandContent, Json::Value& replyContent )
{
  // Ignored
  bool outputIncluded = commandContent.get( "output", false ).asBool();
  bool rawInputIncluded = commandContent.get( "raw", false ).asBool();
  
  int sessionNumber = commandContent.get( "session", 0 ).asInt();
  
  int startLine = commandContent.get( "start", 0 ).asInt();
  int stopLine = commandContent.get( "stop", 0 ).asInt();
  
  setSearchedTokenInScilabHistory( NULL ); // Adds all history to the search
  int historyLength = getSizeAllLinesOfScilabHistory();
  
  std::string accessType = commandContent.get( "hist_access_type", "range" ).asString();  
  if( accessType == "range" )
  {    
    // Preventing out of bounds request range
    if( stopLine >= historyLength ) stopLine = historyLength - 1;
    if( startLine > stopLine ) startLine = stopLine;
  }
  else
  {
    stopLine = historyLength - 1;
    
    if( accessType == "tail" )
    {
      // There is no way to define search interval, so I'm using this value only for "tail" option
      int cellsNumber = commandContent.get( "n", 0 ).asInt();
      startLine = historyLength - cellsNumber;
    }
    else if( accessType == "search" )
    {
      std::string searchToken = commandContent.get( "pattern", "" ).asString();
      bool isUnique = commandContent.get( "unique", false ).asBool();
      
      // Define search token (updates search output list)
      setSearchedTokenInScilabHistory( (char*) searchToken.data() );
      int historySearchLength = getSizeAllLinesOfScilabHistory();
      
      // Allow only the last match to be sent to client
      if( isUnique && historySearchLength > 1 ) historySearchLength = 1; 
      startLine = historyLength - historySearchLength;
    }
  }
  
  // History range/search entries are browsed from the last to the first
  replyContent[ "history" ] = Json::Value( Json::arrayValue );
  for( int historyLineIndex = stopLine; historyLineIndex >= startLine; historyLineIndex-- )
  {
    // It looks like session number is different than session UUID (it's an integer
    // rather than a string). So I'm just assuming it is right and returning the same value
    std::stringstream historyLineStream( "(" );
    historyLineStream << sessionNumber;
    historyLineStream << ",";
    historyLineStream << historyLineIndex;
    historyLineStream << "," << getPreviousLineInScilabHistory() << ")";
    
    replyContent[ "history" ][ historyLineIndex ] = historyLineStream.str();
  }
}
{% endhighlight %}



Well, that was a rather short and quick post, but I hope that it was clear enough. I'll probably update it after there is a resolution for what to do about the missing pieces.


Thanks again for reading. See you next time !!
