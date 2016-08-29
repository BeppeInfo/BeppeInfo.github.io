---
layout: post
title: Post GSoC 2016 - Part I
subtitle: Handling Multiple Users (Almost...)
category: Programming
tags: [GSoC, Scilab, C++, Jupyter]
---     

Hello again ! And before you start wondering, this is not exactly a new **GSoC** project report.

As I stated on my last report, a significant amount of work remains to be done for our **Scilab Jupyter** kernel (maybe even more than I imagine, as [my pull request has not even been reviewed yet](https://codereview.scilab.org/#/c/18489/)). One of the issues that seem more trivial, and, at the same time, more challenging, is support for multiple concurrent users in the same running kernel.

### Problem statement

In practice that would happen when opening a new tab for an already running kernel on Jupyter Notebook...

<p align="center">
  <img src="/img/notebook_open_running.png">
</p>

... or by calling a first instance of e.g. **qtconsole** normally...

{% highlight bash % }
$ jupyter qtconsole --kernel scilab
{% endhighlight %}

... and the subsequent ones with:

{% highlight bash % }
$ jupyter qtconsole --existing
{% endhighlight %}


Partly, the protocol itself enables that support by [defining **"busy"** and **"idle"** status messages](http://jupyter-client.readthedocs.io/en/latest/messaging.html#kernel-status), published to all clients. That way, while some command is being processed, frontends get notified that new requests should wait to be sent, and kernel usage gets multiplexed.

However, two problems remain: completeness verification of multiline (and multi message) commands and variable name sharing.


### Code Completeness Verification

Firstly, what would happen if during issuing of a for loop in a client...

{% highlight octave % }
>> for i=1:10
>>
{% endhighlight %}

Another user finishes writing another control structure ?

{% highlight octave % }
>> if condition then
...
>> end
{% endhighlight %}

As our **JupyterKernel::HandleCompletenessRequest** method currently creates only one static [**parser**](https://en.wikipedia.org/wiki/Parsing), the **end** keyword in the second command would make the parser return a **complete** status, screwing execution handling for the first control structure.

A quick (and, I hope, clever enough) solution for it is replacing the single parser for a static [**map**](http://www.cplusplus.com/reference/map/map/) (also know as [**hash table**](https://en.wikipedia.org/wiki/Hash_table)) data structure, for indexing different code parsers by the unique string identifier of each client, which is provided for every message:

{% highlight cpp % }
void JupyterKernel::HandleCompletenessRequest( JupyterMessage& commandMessage, Json::Value& replyContent )
{
  // Independent parser (not the engine's one) hash table
  // Declared "static" so that previous state is kept
  static std::map<std::string, Parser> checkersTable;
  
  std::string identifier = commandMessage.identifier;
  std::string code = commandMessage.content.get( "code", "" ).asString();
  
  // Thankfully, std::map [] operator already inserts a new element if it doesn't exist
  Parser& checker = checkersTable[ identifier ];
  
  checker.parse( code.data() );     // Verify code without submitting it to the engine
  
  bool isComplete = false;
  replyContent[ "status" ] = "incomplete"; // when "unknown" ?
  if( checker.getControlStatus() == Parser::ControlStatus::AllControlClosed )
  {
    isComplete = true;
    if( checker.getExitStatus() == Parser::ParserStatus::Succeded )
      replyContent[ "status" ] = "complete";
    else
    {
      replyContent[ "status" ] = "invalid";
      checker.cleanup();
    }
  }
  
  // Frontends like QtConsole use their own prompt string and 
  // appends the received one to the code, so this should be empty
  if( not isComplete ) replyContent[ "indent" ] = "";
}
{% endhighlight %}

Now we can store previous state for each client separately.


### Shared Variable Names

Even if we split control flow state management, nothing prevents different clients from declaring variables with the same name/reference. The simplest solution would be to have, as the parser, a [**context**](https://en.wikipedia.org/wiki/Context_(computing)) for every user. However, **Scilab**'s internal **API** doesn't allow the free instancing of new context objects, as it is declared as a **singleton**, that is, contructors and destructor are private methods, indirectly called by static member functions that guarantee creation of at most a single object of that class during the whole process execution:

- From **modules/ast/includes/symbol/context.hxx**
{% highlight cpp %}
#include "function.hxx"
#include "variables.hxx"
#include "libraries.hxx"

extern "C"
{
#include "dynlib_ast.h"
}

namespace symbol
{

/** \brief Define class Context.
*/
class EXTERN_AST Context
{
public:
    typedef std::map<Symbol, Variable*> VarList;
    typedef std::stack<VarList*> VarStack;

    static Context* getInstance(void);

    static void destroyInstance(void);

    /* [...] */

    int getConsoleVarsName(std::list<std::wstring>& lst);
    int getVarsName(std::list<std::wstring>& lst);
    int getMacrosName(std::list<std::wstring>& lst);
    int getFunctionsName(std::list<std::wstring>& lst);

    /* [...] */

private:

    types::InternalType* get(const Symbol& key, int _iLevel);
    bool clearCurrentScope(bool _bClose);
    void updateProtection(bool protect);

    std::list<Symbol>* globals;
    VarStack varStack;
    Variables variables;
    Libraries libraries;
    VarList* console;
    int m_iLevel;

    Context();
    ~Context();

    static Context* me;
};
{% endhighlight %}

- From **modules/ast/includes/symbol/context.cpp**
{% highlight cpp %}
Context* Context::getInstance(void)
{
    if (me == nullptr)
    {
        me = new Context();
    }
    return me;
}

void Context::destroyInstance(void)
{
    if (me)
    {
        delete me;
        me = nullptr;
    }
}
{% endhighlight %}


The usage of a **singleton** actually makes a lot of sense for **Scilab** current shells, as locally running process could have only a user at a time. But it ends up being a roadblock for us.


### So What Now ?

As it turns out, **qtconsole** doesn't seem to handle the **--existing** flag very well, freezing on start and not even showing the command prompt (even for **IPython**). The command-line **console** application currently crashes for the **Scilab** kernel (still wondering why), so we can't test it. Finally, **Jupyter Notebook** doesn't perform command completeness verification, leaving it for the user to determine (**Enter** key increments line, **Shift-Enter** submits the command for execution, regardless of its completeness).

Nevertheless, the issue of context sharing still remains, and I'll have to end up consulting the developers about what could be done about it.

Thanks once again for reading, and until next time !!
