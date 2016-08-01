---
layout: post
title: My 2016 GSoC Project - Part XII
subtitle: Tools of The Trade
category: Programming
tags: [GSoC, Scilab, C++, Autotools]
---     

Hello, again (and quicker than last time) !

When it comes to solving a problem, much is said about the value of knowing precisely which screws to twist, rather than being capable of twisting a lot of screws. But what we normally neglect is that first you need to know how to use a **screwdriver** ! 

Well, it seems like that is a good analogy for software development of a certain magnitude.

**Scilab** surely is a complex project, and even the tools used to automate much of its code development, maintenance and compilation also end up being of a little complex usage. 

At least at first sight.

If you are somehow familiar with **Linux** and building programs from source, you most probably already faced this [maybe infamous] installation pattern on your **terminal**:

{% highlight bash %}
$ ./configure
[...]
$ make
[...]
$ sudo make install
[...]
{% endhighlight %}


The **make** command always look for a file named **Makefile**, located in the current shell folder. Makefiles are something used on many platforms (like **Linux**, **Windows** and **MacOS**) to automate the process of calling the right platform compiler (**gcc**, **clang**, **msvc**, etc.) and linker, setting the compiler/linker flags and listing all the sources and libraries involved on the build of your project. Without it, you would have to define all that stuff by hand every time.

The problem is that writing a **Makefile** could become laborious and error prone, as your project gets to big or subject to some variability (**"What if I could have implementations X and Y of the same library interface"**). 

That's when tools to automatically generate those build files come to rescue, being almost mandatory if one wishes to keep its sanity.

Maybe you already heard about projects like **CMake**, **SCons**, **qmake** or whatever, which are cross-platform and somehow modern programs intended for that purpose. But before them, on **Linux**, there was **autotools**.

The **configure** script we run before the actual building (**make**) process is the **autotools** **Makefile** generator. Which is in turn generated from the **configure.ac** definition file.

*Seriously ?*

<p align="center">
  <img src="https://cdn.meme.am/instances/500x/70042692.jpg">
</p>

Actually, there is more to it.

**Autotools** is a set of 3 different applications, used in sequence before and during the building operations. In each step, simpler input configuration files is used to generate more complex and specific ones.

- **automake** takes all **Makefile.am** handwritten files listed in **configure.ac** and generates corresponding **Makefile.in** files.
- **autoconf** takes **configure.ac** to generate the **configure** script, used to produce a **Makefile** for each **Makefile.in**.
- **libtool** is used during compilation (running **make**) to build the internal project libraries, if any.

Learning how those work, at least superficially, was essential for adding my **jupyter** module to the whole **Scilab**'s build. 

Firstly, each module folder contains its own **Makefile.am**, for creating the module's internal libray. I copied as little as needed (or slightly more than that) from other existing modules to write mine:

- **SCI/modules/jupyter/Makefile.am**
{% highlight bash %}
### SOURCES ###
JUPYTER_CPP_SOURCES = \
    src/cpp/InitializeJupyter.cpp src/cpp/jsoncpp.cpp src/cpp/JupyterKernel.cpp \
    src/cpp/JupyterKernelConnection.cpp src/cpp/JupyterMessage.cpp \
    src/cpp/JupyterMessageHash.cpp src/cpp/JupyterMessagePrint.cpp \
    src/cpp/JupyterMessageRead.cpp src/cpp/uuid.cpp

# Includes need for the compilation
libscijupyter_la_CPPFLAGS = \
    -I$(top_srcdir)/modules/api_scilab/includes -I$(srcdir)/includes/ -I$(srcdir)/src/cpp/ \
    -I$(top_srcdir)/modules/ast/includes/ast/ -I$(top_srcdir)/modules/ast/includes/exps/ \
    -I$(top_srcdir)/modules/ast/includes/operations/ -I$(top_srcdir)/modules/ast/includes/parse/ \
    -I$(top_srcdir)/modules/ast/includes/symbol/ -I$(top_srcdir)/modules/ast/includes/system_env/ \
    -I$(top_srcdir)/modules/ast/includes/types/ -I$(top_srcdir)/modules/jvm/includes/ \
    -I$(top_srcdir)/modules/action_binding/includes -I$(top_srcdir)/modules/history_manager/includes/ \
    -I$(top_srcdir)/modules/ui_data/includes/ -I$(top_srcdir)/modules/threads/includes \
    -I$(top_srcdir)/modules/completion/includes -I$(top_srcdir)/modules/output_stream/includes \
    -I$(top_srcdir)/modules/string/includes -I$(top_srcdir)/modules/fileio/includes \
    -I$(top_srcdir)/modules/localization/includes -I$(top_srcdir)/modules/commons/src/jni \
    -I$(top_srcdir)/modules/dynamic_link/includes \
    $(AM_CPPFLAGS)

# Name of the library
pkglib_LTLIBRARIES = libscijupyter.la

# All the sources needed by libscijupyter.la
libscijupyter_la_SOURCES = $(JUPYTER_CPP_SOURCES)

libscijupyter_la_LDFLAGS = $(AM_LDFLAGS) $(JUPYTER_LIBS)
libscijupyter_la_LIBADD = $(JUPYTER_LIBS)

#### Target ######
modulename=jupyter

#### jupyter : include files ####
libscijupyter_la_includedir=$(pkgincludedir)
libscijupyter_la_include_HEADERS = includes/dynlib_jupyter.h includes/InitializeJupyter.h \
                                   includes/JupyterMessagePrint.h includes/JupyterMessageRead.h

# Common definitions added to every module
include $(top_srcdir)/Makefile.incl.am
{% endhighlight %}


Then, we include the new file in the **configure.ac** list. As the **jupyter** module addition would require extra dependencies (**ZeroMQ** and native **UUID** library), it's possible to allow its disablement by creating a new configuration option, and check for their existance in the running operating system.

- **SCI/configure.ac**
{% highlight bash %}
[...]
## New configure option
AC_ARG_ENABLE(jupyter,
    AS_HELP_STRING([--enable-jupyter],[Build the Scilab Jupyter kernel binary]))
[...]
## Now automake will look the corresponding Makefile,am
AC_CONFIG_FILES([ ... modules/jupyter/Makefile ... ])
[...]
###########################
########## Jupyter checks
###########################
## Check for dependencies of Jupyter kernel functionality

if test "$enable_jupyter" = yes; then
    AC_CHECK_LIB([zmq], [zmq_ctx_new], [JUPYTER_LIBS=" -lzmq"], 
                  [AC_MSG_ERROR([ZeroMQ not found. Use --disable-jupyter or install ZeroMQ library])])
    AC_CHECK_LIB([uuid], [uuid_is_null], [JUPYTER_LIBS+=" -luuid"], 
                  [AC_MSG_ERROR([uuid not found. Use --disable-jupyter or set the path to uuid library])])
    
    AC_SUBST(JUPYTER_LIBS)
fi

## Set JUPYTER condition to TRUE
AM_CONDITIONAL(JUPYTER, test "$enable_jupyter" = yes)
[...]
{% endhighlight %}


After that, we conditionally add our module library to the linking process:

- **SCI/modules/Makefile.am**
{% highlight bash %}
[...]
if JUPYTER
SUBDIRS += jupyter
endif
[...]
if JUPYTER
ENGINE_LIBS += $(top_builddir)/modules/jupyter/libscijupyter.la
endif
[...]
{% endhighlight %}


Finally, we add a new executable target to the base **Makefile.am**, from now, based on the **scilab-cli-bin** one (no graphical features).

- **SCI/Makefile.am**
{% highlight bash %}
[...]
bin_PROGRAMS		= scilab-bin scilab-cli-bin

bin_SCRIPTS			= bin/scilab bin/scilab-adv-cli bin/scilab-cli \
bin/scinotes bin/xcos 

if IS_MACOSX
bin_SCRIPTS 		+= bin/checkmacosx.applescript
endif

if JUPYTER
bin_PROGRAMS 		+= jupyter-scilab-kernel-bin
bin_SCRIPTS 		+= bin/jupyter-scilab-kernel
endif

scilab_bin_LDFLAGS      = $(AM_LDFLAGS) $(OPENMPI_LIBS)
scilab_cli_bin_LDFLAGS  = $(AM_LDFLAGS) $(OPENMPI_LIBS)
[...]
if JUPYTER
scilab_bin_LDADD += $(top_builddir)/modules/jupyter/libscijupyter.la $(JUPYTER_LIBS)
jupyter_scilab_kernel_bin_SOURCES = modules/startup/src/cpp/jupyter_scilab_kernel.cpp
jupyter_scilab_kernel_bin_CPPFLAGS = $(scilab_cli_bin_CPPFLAGS) -I$(top_srcdir)/modules/jupyter/includes/
jupyter_scilab_kernel_bin_LDADD = $(scilab_cli_bin_LDADD)
endif
[...]
{% endhighlight %}

