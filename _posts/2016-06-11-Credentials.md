---
layout: post
title: My 2016 GSoC Project - Part VII
subtitle: May we see your credentials ?
category: Programming
tags: [GSoC, Jupyter]
---  

Quicker than last time, I'm back.

Being the kind of guy who always developed software projects on my own, coding for study or other personal purposes, sometimes focusing on few people (who end up not using it) besides myself, I never worried that much about making my applications provide execution safety or data integrity guarantees. Adding too many verifications to the code always looked like complexifying it and adding overhead for the sake of that **0.001%** of the cases.

Well, as software made for a much broader audience, on **Jupyter** systems those **0.001%** could be significant, enough to justify the implementation of mechanisms for ensuring consistency of that big amount of messages traveling across processes. Not being a performance critical application (at least not the communication part), I guess the overhead could be neglected.

So, this time I couldn't escape taking safety into account and had to program like the big boys, learning **Jupyter**'s way to do it...


**Maaaan, it was a hard path !!** 


Much more than it should have been, I must say, because what **Jupyter** defines for its applications (clients and kernels) to do is quite simple: we need to generate a [**hash**](https://en.wikipedia.org/wiki/Hash_function) based on the data (actually, **Header**, **Parent Header**, **Metadata** and **Content** frames) of the message.

The difference from a hash to a simple unique identifier (like [**UUID**](https://en.wikipedia.org/wiki/Universally_unique_identifier)), that we use as well, is that the hash is calculated by a given algorithm that operates over the input data (ou 4 frames, in this case), so that we always get the same output for the same input. That's a form of irreversible encryption for integrity verification: we pass the calculated hash along with the proper data and run the algorithm again on the other to see if the same result is generated (obviously, if not, something went wrong).

By default, **Jupyter** asks for a **keyed-hash message authentication code** ([**HMAC**](https://en.wikipedia.org/wiki/Hash-based_message_authentication_code)) calculated with the [**SHA256**](https://en.wikipedia.org/wiki/SHA-2) hash function to verify each message. I say "by default" because the algorithm to be used is defined in the **"signature_scheme"** field of our **JSON** configuration file, so I suppose that other functions are allowed (probably other **SHA** variants), but I didn't find information about which ones.

{% highlight json %}
# file /run/user/1000/jupyter/kernel-9617.json
{
  "kernel_name": "scilab",
  "transport": "tcp",
  "hb_port": 36040,
  "signature_scheme": "hmac-sha256",
  "control_port": 57994,
  "key": "0abfec75-90fd-4a0e-ad2e-f8125118b3a9",
  "ip": "127.0.0.1",
  "shell_port": 44010,
  "stdin_port": 58295,
  "iopub_port": 36388
}
{% endhighlight %}

Here, **"key"** is the value used to initialize every hash calculation, and changes each time we run our program (you can force it to be an empty string, to disable authentication alltogether). Besides, that was the major source of my frustration on the past few days, lol. More on that later...


Once again "according to **Jupyter Docs**", the **hmac-sha256** hash calculation is performed this way, on **IPython**:

{% highlight python %}
import hmac
import hashlib

# once:
digester = HMAC(key, digestmod=hashlib.sha256)

# for each message
d = digester.copy()
for serialized_dict in (header, parent, metadata, content):
    d.update(serialized_dict)
signature = d.hexdigest()
{% endhighlight %}


Well, I'm not coding in **Python** now, and **C++** doesn't have standard libraries for this. However, as you would expect from me, I didn't even try to make the calculations by hand, instantly looking for an existing implementation.

The first solution I found was the [**OpenSSL**](https://www.openssl.org/) project **HMAC** libray, but I didn't quite like it, because:

- As much as I am fond of **C**, if I'm using **C++** anyways, I prefer to use **C++** **API**s. Sorry for the weak argument.
- **OpenSSL** is a big colection of functionalities for performing secure connections. Pulling all of these dependencies just to use a few functions of its cryptography subset seems like bloating the application.
- It's weird to use, requires many calls and conversions between native data types and classes. Again, I would gladly afford to use it in a pure **C** program, but not in this case.

Resuming my search, I quickly came across [**Crypto++**](http://www.cryptopp.com/) (also known as **CryptoPP**, **libcrypto++**, and **libcryptopp**). It provides just cryptography functions with nice object-oriented abstractions to facilitate the job. That was my choice.



Things were going rather smooth up to that moment, but they went off the rails when I started assuming things by myself.

As you can see, the **HMAC** key provided by **Jupyter**'s configuration is an hexadecimal format string, which is taken unchanged by **Python**'s **HMAC** object (**Digester**) constructor. On the other hand, both **OpenSSL** and **Crypto++** documentation stated that their **Digester** initialization **key** parameter should be a byte array/sequence. That apparent difference made me begin trying to find ways to convert from one representation to the other, something like:

>'c9dbf6fd-3fd1-4185-8648-67b101e5785d' -> [0xC9, 0xDB, 0xF6, 0xFD, 0x3F, 0xD1, 0x41, 0x85, 0x86, 0x48, 0x67, 0xB1, 0x01, 0xE5, 0x78, 0x5D]

I was successful in doing so, at least with **Crypto++**, as it provided an easy way. But the fact is that the generated hash wasn't the same as the **Python** implementation one, and I was successively getting errors when sending messages from the kernel to the **Jupyter** console:

{% highlight python %}
ERROR:tornado.application:Exception in callback None
Traceback (most recent call last):
  File "/usr/lib/python3.5/site-packages/tornado/ioloop.py", line 883, in start
    handler_func(fd_obj, events)
  File "/usr/lib/python3.5/site-packages/tornado/stack_context.py", line 275, in null_wrapper
    return fn(*args, **kwargs)
  File "/usr/lib/python3.5/site-packages/zmq/eventloop/zmqstream.py", line 440, in _handle_events
    self._handle_recv()
  File "/usr/lib/python3.5/site-packages/zmq/eventloop/zmqstream.py", line 472, in _handle_recv
    self._run_callback(callback, msg)
  File "/usr/lib/python3.5/site-packages/zmq/eventloop/zmqstream.py", line 414, in _run_callback
    callback(*args, **kwargs)
  File "/usr/lib/python3.5/site-packages/tornado/stack_context.py", line 275, in null_wrapper
    return fn(*args, **kwargs)
  File "/usr/lib/python3.5/site-packages/jupyter_client/threaded.py", line 88, in _handle_recv
    msg = self.session.deserialize(smsg)
  File "/usr/lib/python3.5/site-packages/jupyter_client/session.py", line 859, in deserialize
    raise ValueError("Invalid Signature: %r" % signature)
ValueError: Invalid Signature: b'fa7ddb30b91eeaeb95b1c07790682c5bdc7f5c03ffd332d88773ffa2de3ae70bu'
{% endhighlight %}


I struggled with it for some time, trying to find the source of the problem. The generated hash was getting wrong even without new data added after the **Digester** creation, so the key processing was wrong.

After many attempts, tired and frustrated, I plainly tried to pass the key string as the byte sequence.... And it worked !!!

Jeez... I felt like a complete idiot for not trying the simplest thing before any other...


All in all, I ended up encapsulating the hash value generation in a simple class, that you can see below:

{% highlight cpp %}
#include <cryptopp/hex.h>
#include <cryptopp/hmac.h>
#include <cryptopp/sha.h>
#include <algorithm>
#include <string>

class MessageHashGenerator
{
public:
  static void SetKey( std::string keyString, std::string& hashAlgorithm )
  {
    if( keyString.empty() )
    {
      enabled = false;
      return;
    }
    
    hashGenerator.SetKey( (const byte*) keyString.data(), keyString.size() );
    
    enabled = true;
  }
  
  static std::string GenerateHash( std::string& header, std::string& parent, std::string& metadata, std::string& content )
  {
    CryptoPP::SecByteBlock hash( 32 );
    std::string hashString;
    
    if( enabled )
    {
      // Clear digester internal value and start adding new message data
      hashGenerator.Restart();
      hashGenerator.Update( (const byte*) header.data(), header.size() );
      hashGenerator.Update( (const byte*) parent.data(), parent.size() );
      hashGenerator.Update( (const byte*) metadata.data(), metadata.size() );
      hashGenerator.Update( (const byte*) content.data(), content.size()  );
      hashGenerator.Final( hash.data() );
    
      // Stream concatenation to generate hexadecimal format string from the hash byte sequence
      CryptoPP::StringSource( hash.data(), hash.size(), true, new CryptoPP::HexEncoder( new CryptoPP::StringSink( hashString ) ) );
      // String is uppercase. Convert to lowercase
      std::transform( hashString.begin(), hashString.end(), hashString.begin(), ::tolower );
    }
    
    return hashString;
  }
  
private:
  static CryptoPP::HMAC<CryptoPP::SHA256> hashGenerator;
  static bool enabled;
};

// Static variables initialization
CryptoPP::HMAC<CryptoPP::SHA256> MessageHashGenerator::hashGenerator;
bool MessageHashGenerator::enabled = false;
{% endhighlight %}

(Take it easy, I'll put definition on a proper source file later. I promise...)


Having successfully added **HMAC** signature, together with [**ISO** format time stamp](https://en.wikipedia.org/wiki/ISO_8601) and random **UUID**, now we have all identification and authentication data common to all **Jupyter** messages, and I can move forward to implement the remaining specific bits.


That's all for now. See ya later !!
