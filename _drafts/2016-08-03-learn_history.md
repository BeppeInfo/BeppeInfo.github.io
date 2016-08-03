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
{% highlight json %}
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
{% highlight json %}
content = {
  # A list of 3 tuples, either:
  # (session, line_number, input) or
  # (session, line_number, (input, output)),
  # depending on whether output was False or True, respectively.
  'history' : list,
}
{% endhighlight %}


Basically, one can request a range or the entries that match a given pattern from the list of recorded previous commands (and their respective outputs).
