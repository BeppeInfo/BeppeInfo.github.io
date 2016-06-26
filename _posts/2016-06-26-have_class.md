---
layout: post
title: My 2016 GSoC Project - Part IX
subtitle: Please, have some Class...
category: Programming
tags: [GSoC, Jupyter, ZeroMQ, C++]
---  

Greetings everyone !

As I said before, **GSoC Mid-Term** phase is on, and, before it ends, I really wish to be [mostly] done with the **ZeroMQ** connections, **Jupyter** messaging and **HMAC** authentication part of my project. Obviously I'll need to come back to it sometimes to make ajustments and implement some missing functionality, but my work should be focused on the **Scilab** side of things from now on.

I really should be thankful that my mentors understand that my output have been kinda low up till now due to academic work, since vacation haven't started yet (by the way, "vacation" is a somewhat inaccurate term for post graduate students, lol), and that I kept myself busy doing some groundwork. But they made it clear that I need to show a **Scilab** kernel implementation working, and I do agree. I'm hopeful that from now I can have more time to do my job and compensate for delays.

Not that I haven't done anything in the past weeks. As I said, I want to get rid of my concerns about what I have worked on so far, and my code was quite hacky or loosely implemented. So I dedicated myself to refactor it on a more object-oriented structure, spliting its semantically different pieces into classes. Having it that way will help me adding the next components, I hope.

<p align="center">
  <img src="http://cdn.meme.am/instances/42651831.jpg">
</p>

Basically, I defined **ProtocolMessage** and **ProtocolConnection** classes, dealing with **Jupyter** message structure and the needed set of **ZeroMQ** sockets, respectively. My intention was to let the **main** code see only a connection object and a queue of incoming and outgoing message objects, from which it takes or sets only the **JSON** data relevant to the kernel language engine (**Scilab** libraries, in our case).

I think the best way to explain my idea is showing you at least the classes headers. So there they go.

- **protocol_message.hpp** (header guards omitted)

{% highlight cpp %}
// JsonCpp functions
#include <json/json.h>  
#include <json/reader.h>
#include <json/writer.h>

// For ISO format time stamp string
#include <ctime>
#include <cstring>
#include <sys/time.h>

// Platform specific unique identifier generator
#include "uuid.h"

#include <string>
#include <iostream>

class ProtocolMessage
{
public:
  // The first argument is used connection's unique identifier.
  // Second and third are always set to "kernel" and "5.0" for now.
  ProtocolMessage( std::string sessionUUID, std::string userName, std::string version );
  
  // Returns internal message JSON data in serialized string form
  void SerializeData( std::string& serialHeader, std::string& serialParent, 
                      std::string& serialMetadata, std::string& serialContent );
  // Takes serialized string data and updates its internal JSON objects
  void DeserializeData( std::string& serialHeader, std::string& serialParent, 
                        std::string& serialMetadata, std::string& serialContent );
  
  // Returns "msg_type" field of message header
  std::string GetType();
  // Defines "msg_type" field and sets the rest of message header accordingly
  void SetType( std::string messageType );
  
  // Use native date functions to generate a ISO format time string for the message
  void UpdateTimeStamp();
  
  // Holds client info
  std::string identifier;
  
  // Data used by the kernel implementation
  Json::Value metadata, content;
  
  // Initialized to "<IDS|MSG>". Common to all messages.
  static const char* DELIMITER;
  
private:  
  std::string sessionUUID;
  std::string userName;
  std::string versionNumber;
  Json::Value header, parent;
  Json::FastWriter serializer;
  Json::Reader deserializer;
  
  // Utility function
  void BuildNewHeader();
};
{% endhighlight %}

- **protocol_connection.hpp** (header guards omitted)

{% highlight cpp %}
#include "protocol_message.hpp"
#include "message_hash_generator.hpp"
#include "uuid.h"

#include <zmq.hpp>      // ZeroMQ functions

#include <thread>       // C++11 threads
#include <chrono>       // C++11 time handling functions

#include <string>
#include <queue>
#include <iostream>
#include <string>

class ProtocolConnection
{
public:
  // Constructor takes most of the information from Jupyter kernel config file
  ProtocolConnection( std::string transport, std::string ipHost, 
                      std::string ioPubPort, std::string controlPort, 
                      std::string shellPort, std::string stdinPort, 
                      std::string heartbeatPort );
  
  ~ProtocolConnection();
  
  // Define hash key and algorithm for the HMAC engine
  void SetHashGenerator( std::string keyString, std::string signatureScheme );
  
  // Put all available incoming messages in a queue in order
  // of priority (e.g. control socket messages first)
  std::queue<ProtocolMessage>& ReceiveMessages();
  // Redirects each message to its proper socket according to "msg_type" field
  void SendMessage( std::queue<ProtocolMessage>& messageOutQueue );
  
  // Stops heartbeat thread
  void Shutdown();
  
private:
  // Context is shared by all connections in the same process
  static zmq::context_t context;
  
  std::string sessionUUID;
  zmq::socket_t ioPubSocket, controlSocket, stdinSocket, shellSocket;
  MessageHashGenerator messageHashGenerator;
  std::queue<ProtocolMessage> messageInQueue;
  bool isRunning;
  
  // Poller for detecting incoming Stdin, Shell and Control request messages
  zmq::pollitem_t requestsPoller[ 4 ];
  
  // Helper functions
  bool ReceiveProtocolMessage( zmq::socket_t* receivingSocket, ProtocolMessage& message );
  void SendProtocolMessage( zmq::socket_t* sendingSocket, ProtocolMessage& message );
  
  // Heartbeat thread and function declaration
  std::thread heartbeatThread;
  // Method is static because we can't pass a member function pointer to std::thread.
  // So we pass the connection running state to a non member thread function
  static void RunHeartbeatLoop( std::string& transport, std::string& ipHost, 
                                std::string& port, bool* ref_isRunning );
};
{% endhighlight %}


If any of you want, I can detail the implementation better in the comments or later on a post update.

See you next !!
