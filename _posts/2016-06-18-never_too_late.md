---
layout: post
title: My 2016 GSoC Project - Part VIII
subtitle: It's never too late to go back
category: Programming
tags: [GSoC, Jupyter, Scilab, OpenSSL]
---  

Ok, I know I'm late again this week, but that is not (completely) due to laziness. I'd wish to say that it has to do with **real life issues**, but I'm not sure if academic life can be called **"real life"** (or even **"life"**). This routine is really taking a toll on me.

To make it brief, I just want to let you know that some things stated on my last post are not valid anymore: talking to **Clément David** (if I have not mentioned him yet, he is one of my **GSoC** mentors) at [Scilab GSoC mailing list](http://mailinglists.scilab.org/Scilab-GSOC-Mailing-Lists-Archives-f2646148.html), I got to know that **OpenSSL** is already a dependency of **Scilab**, so to run my **Jupyter Kernel** it'll be needed anyways.

That detail changes our previous scenario, as now only using **Crypto++** for **HMAC** hash calculation has the disavantage of adding an extra dependency, which is undesired.

For this reason, I ended up needing to rewrite the hash generator class to use **OpenSSL** **HMAC** **C** API. It wasn't that difficult, as the logic behind the calculations remains pretty much the same.

Here is the resulting code:

{% highlight cpp %}
#include <openssl/hmac.h>
#include <string>
#include <cstring>
#include <iostream>

class MessageHashGenerator
{
public:
  MessageHashGenerator()
  {
    digestEngine = NULL;
    enabled = false;
  }
  
  bool SetKey( std::string keyString, std::string hashAlgorithm = "hmac-sha256" )
  {
    OpenSSL_add_all_digests();    // Enables the search for encryption engines
    
    enabled = false;
    
    key = keyString;
    
    if( hashAlgorithm.find( "hmac-" ) != std::string::npos )
    {
      std::string digestEngineName = hashAlgorithm.substr( hashAlgorithm.find_last_of( "-" ) + 1 );
      
      // Search for the encryption engine by its name
      if( (digestEngine = EVP_get_digestbyname( digestEngineName.c_str() )) != NULL )
        enabled = true;
    }
    
    return enabled;
  }
  
  std::string GenerateHash( std::string& header, std::string& parent, std::string& metadata, std::string& content )
  {
    static HMAC_CTX digestContext;
    unsigned char hash[ 32 ];
    unsigned int hashLength;
    char hashString[ 65 ] = "";   // 64 characters + NULL terminator
    
    if( enabled )
    {
      HMAC_Init_ex( &digestContext, key.data(), key.size(), digestEngine, NULL );
      HMAC_Update( &digestContext, (const unsigned char*) header.data(), header.size() );
      HMAC_Update( &digestContext, (const unsigned char*) parent.data(), parent.size() );
      HMAC_Update( &digestContext, (const unsigned char*) metadata.data(), metadata.size() );
      HMAC_Update( &digestContext, (const unsigned char*) content.data(), content.size() );
      HMAC_Final( &digestContext, hash, &hashLength );
      HMAC_CTX_cleanup( &digestContext );
    
      // Each byte is represented by 2 characters
      for( size_t byteIndex = 0; byteIndex < hashLength; byteIndex++ )
        sprintf( hashString + strlen( hashString ), "%02x", hash[ byteIndex ] );
    }
    
    return hashString;
  }
  
private:
  std::string key;
  const EVP_MD* digestEngine;
  bool enabled;
};
{% endhighlight %}


That's all for now folks. I'll come back soon !
