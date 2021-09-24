
# zproto - a protocol framework for ZeroMQ

## Contents


**[Man Page](#man-page)**

**[The Codec Generator](#the-codec-generator)**

**[The Server Generator](#the-server-generator)**

**[Quick Background](#quick-background)**

**[The State Machine Model](#the-state-machine-model)**

**[The zproto Server Model](#the-zproto-server-model)**

**[Superstates](#superstates)**

**[Client and Server Properties](#client-and-server-properties)**

**[Client Connection Expiry](#client-connection-expiry)**

**[Server Configuration File](#server-configuration-file)**

**[CZMQ Reactor Integration](#czmq-reactor-integration)**

**[The Client Generator](#the-client-generator)**

**[The zproto Client Model](#the-zproto-client-model)**

**[Heartbeating](#heartbeating)**

**[Superstates](#superstates)**

**[Before/After State Actions](#beforeafter-state-actions)**

**[Client Properties](#client-properties)**

**[Client Expiry Timer](#client-expiry-timer)**

**[Method Framework](#method-framework)**

**[Custom hand-written methods](#custom-hand-written-methods)**

**[Protocol Design Notes](#protocol-design-notes)**

**[Heartbeating and Client Expiry](#heartbeating-and-client-expiry)**

**[For More Information](#for-more-information)**

**[This document](#this-document)**

## Man Page

zproto is a set of code generators that can produce:

* fast and efficient binary codecs for ZeroMQ-based protocols.

* full-featured protocol servers based on high-level state machine models.

* full-featured protocol clients based on high-level state machine models.

To use zproto, clone the repository at https://github.com/zeromq/zproto.

Build and test using the usual commands:

    ./autogen.sh
    ./configure
    make check

And then install the code generators:

    make install

Next, read the src/zproto_example.xml file to learn how to write your own protocol specifications. The binary codec has the same name, and is src/zproto_example.c and include/zproto_example.h.

To rebuild the codec, first build and install https://github.com/zeromq/gsl. Then run these commands:

    cd src
    make code check

To use zproto as the base for your own projects, create a new project with [zproject](http://github.com/zeromq/zproject) which nicely integrates with zproto.

## The Codec Generator

Goals of the codec generator:

* Best performance on high-volume low-complexity data.
* Full flexibility on often-changing data (headers).
* Portable to any programming language.
* Built for ZeroMQ.
* Easy to use.

The origin of this project is [Chapter 7 of the ZeroMQ Guide](http://zguide.zeromq.org/page:all#Serializing-Your-Data).

To use this tool, please build and install the [GSL code generator](https://github.com/zeromq/gsl).

To contribute patches back to this code, please send GitHub pull requests, adding your name to AUTHORS. The full contract for contributions is ZeroMQ RFC 22, http://rfc.zeromq.org/spec:22, with the change of using the MIT license.

To use:

* Write your protocol as an XML file, using src/zproto_example.xml as a starting point.
* Generate your protocol, using src/generate as a starting point.
* Add the generated .h and .c files to your git repository.
* Don't modify generated codecs. Change the model, and regenerate.

## The Server Generator

While ZeroMQ gives you a powerful communications engine to use in many different ways, building a conventional server is still fairly heavy work. We use the ROUTER socket for that, managing the state for each individual client connection. The zproto project includes a tool that generates whole servers, in C, from state machine models.

### Quick Background

By Pieter Hintjens.

zproto is based Chapter 7 of my ZeroMQ book, originally used in FileMQ, Zyre, and several other projects.

My company iMatix used to do code generation as our main business. We got... very good at it. There are lots of ways to generate code, and the most powerful and sophisticated code generator ever built by mankind lives on Github.com at [imatix/gsl](https://github.com/imatix/gsl). (Development continues at [zeromq/gsl](https://github.com/zeromq/gsl).) It's an interpreter for programs that eat models (self-describing documents) and spew out text of any shape and form.

The only problem with sophisticated magic like GSL is that it quickly excludes other people. So in ZeroMQ I've been very careful to not do a lot of code generation, only opening that mysterious black box when there was real need.

The first case was in CZMQ, to generate the classes for ZeroMQ socket options. Then in CZMQ, to [generate project files](https://github.com/zeromq/czmq/tree/master/model) (for various build systems) from [a single project.xml](https://github.com/zeromq/czmq/blob/master/src/project.xml) file. Yes, we still use XML models. It's actually a good use case for simple schema-free XML.

Then I started generating binary codecs for protocols, [starting with FILEMQ](https://github.com/zeromq/filemq/blob/master/src/fmq_msg.xml). We used these codecs for a few different projects and they started to be quite solid. Something like a protobufs for ZeroMQ. You can see that the generated code [looks as good](https://github.com/zeromq/filemq/blob/master/include/fmq_msg.h) as hand-written code. It's actually better: [more consistent, with fewer errors](https://github.com/zeromq/filemq/blob/master/src/fmq_msg.c).

Finally, I drew back on an even older iMatix speciality, which was state machines. My first free software tool was [Libero](http://legacy.imatix.com/html/libero/), a great tool for designing code as state machines and producing lovely, robust engines in pretty much any language. Libero predates GSL, so isn't as flexible. However it uses a very elegant and simple state-event-action model.

The Libero model is especially good at designing server-side logic, where you want to capture the exact state machine for a client connection, from start to end. This happens to be one of the heavier problems in ZeroMQ architecture: how to build capable protocol servers. We do a lot of this, it turns out. You can do only so much with low-level patterns like pub-sub and push-pull. Quite soon you need to implement stateful services.

So this is what I made: a GSL code generator that takes a finite-state machine model inspired by Libero, and turns out a full working server. The current code generator produces C (that builds on CZMQ). In this article I'll explain briefly how this works, and how to use it.

### The State Machine Model

State machines are a little unusual, conceptually. If you're not familiar with them it'll take a few days before they click. The Libero model is fairly simple and high level, meant to be designed and understood by humans:

* The machine exists in one of a number of named *states*. By convention the machine starts in the first state.

* In each state, the machine accepts one of a set of named *events*. Unhandled events are either ignored or cause the machine to die, depending on your taste.

* Given an event in a state, the machine executes a list of *actions*, which correspond to your code.

* After executing the actions the machine moves to the *next state*. An empty next state means "stay in same state".

* In the next state, the machine either continues with an *internal event* produced by the previous actions, or waits for an *external event* coming as a protocol command.

* Any action can set an *exception event* that interrupts the flow through the action list and to the next state.

### The zproto Server Model

The zproto_server_c.gsl code generator outputs a single .inc file called an *engine* that does the hard work. If needed, it'll also generate you a skeleton .c file for your server, which you edit and build. It doesn't re-create that file again, though it will append new action stubs.

The server is a "actor" built on the CZMQ/zactor class. CZMQ zactors use a simple, consistent API based on message passing:

    zactor_t *server = zactor_new (some_server, "myserver");
    zstr_send (server, "VERBOSE");
    zstr_sendx (server, "BIND", endpoint, NULL);
    ...
    zactor_destroy (&server);

Where "myserver" is used in logging. Note that a zactor is effectively a background thread with a socket API, and you can pass zactor_t instances to all CZMQ message passing methods. The generated zactor accepts these messages:

    VERBOSE
    LOAD configfile
    SET configpath value
    SAVE configfile
    BIND localendpoint
    $TERM

Rather than run the server as a main program, you write a main program that creates and works with server actors. These run as background services, accepting clients on a ZMQ ROUTER port. The bind method exposes that port to the outside world.

Your input to the code generator is two XML files (without schemas, DTDs, entity encodings!) that defines a set of 'states', and the protocol messages as used to generate the codec. Here is a minimal 'hello_server.xml' example that defines a Hello, World server:

    <class
        name = "hello_server"
        title = "Hello server"
        script = "zproto_server_c"
        protocol_class = "hello_msg"
        protocol_dir = ""
        package_dir = "../include"
        source_dir = "."
        project_header = "hello.h"
        >
        A Hello, World server

        <state name = "start">
            <event name = "HELLO">
                <action name = "send" message = "WORLD" />
            </event>
        </state>
    </class>

You will also need a minimal 'hello_msg.xml' as below. It's name is defined line 5 of 'hello_server.xml', as 'protocol_class'. Variable 'protocol_dir' allows developer to split protocol from client/server implementation to different projects.

    <class
        name = "hello_msg"
        signature = "0"
        title = "hello msg protocol"
        script = "zproto_codec_c"
        package_dir = "../include"
        source_dir = "."
        >

        <message name = "HELLO" />
        <message name = "WORLD" />
    </class>

Names of states, events, and actions are case insensitive. By convention however we use uppercase for protocol events. Protocol events can also contain embedded spaces or hyphens, which are mapped to underscores. In this case, HELLO and WORLD are two messages that must be defined in the hello_msg.xml file.

The generated server manages clients automatically. To build this, do:

    gsl -q -trace:1 hello_server.xml

The first time you do this, you'll get a hello_server.c source file. You can edit that; it won't be regenerated. The generated code goes, instead, into hello_server_engine.inc. Take a look if you like.

The trace option shows all protocol messages received and sent.

There are two predefined actions: "send", which sends a specific protocol message, and "terminate", which ends the client connection. Other actions map to functions in your own code, e.g.:

    <state name = "start">
        <event name = "HELLO" next = "start">
            <action name = "tell the user hello too" />
            <action name = "send" message = "WORLD" />
        </event>
    </state>

    ...

    static void
    tell_the_user_hello_too (client_t *self)
    {
        puts ("Hello, World!");
    }

Your server code (the actions) gets a small API to work with:

    //  Set the next event, needed in at least one action in an internal
    //  state; otherwise the state machine will wait for a message on the
    //  router socket and treat that as the event.
    static void
    engine_set_next_event (client_t *self, event_t event);

    //  Raise an exception with 'event', halting any actions in progress.
    //  Continues execution of actions defined for the exception event.
    static void
    engine_set_exception (client_t *self, event_t event);

    //  Set wakeup alarm after 'delay' msecs. The next state should
    //  handle the wakeup event. The alarm is cancelled on any other
    //  event.
    static void
    engine_set_wakeup_event (client_t *self, size_t delay, event_t event);

    //  Execute 'event' on specified client. Use this to send events to
    //  other clients. Cancels any wakeup alarm on that client.
    static void
    engine_send_event (client_t *self, event_t event);

    //  Execute 'event' on all clients known to the server. If you pass a
    //  client argument, that client will not receive the broadcast. If you
    //  want to pass any arguments, store them in the server context.
    static void
    engine_broadcast_event (server_t *server, client_t *client, event_t event);

    //  Set log file prefix; this string will be added to log data, to make
    //  log data more searchable. The string is truncated to ~20 chars.
    static void
    engine_set_log_prefix (client_t *client, const char *string);

    //  Set a configuration value in the server's configuration tree.
    //  The properties this engine uses are: server/timeout, and
    //  server/background. You can also configure other abitrary properties.
    static void
    engine_configure (server_t *server, const char *path, const char *value);

### Superstates

Superstates are a shorthand to reduce the amount of error-prone repetition in a state machine. Here is the same state machine using a superstate:

    <state name = "start" inherit = "defaults">
        <event name = "HELLO" next = "waiting">
            <action name = "wait on other internal event" />
        </event>
    </state>

    <state name = "waiting" inherit = "defaults">
        <event name = "ok" next = "start">
            <action name = "tell the user hello too" />
            <action name = "send" message = "WORLD" />
        </event>
        <event name = "error" next = "start">
            <action name = "terminate" />
        </event>
    </state>

    <state name = "defaults">
        <event name = "PING">
            <action name = "send" message = "PONG" />
        </event>
    </state>

Note the logic of PING, which says, "when the client sends PING, respond by sending PONG, and then stay in the same state."

For complex protocols you can collect error handling together using the wildcard event, "*", which means "all other protocol events in this state". For example:

    <state name = "defaults">
        <event name = "PING">
            <action name = "send" message = "PONG" />
        </event>
        <event name = "*">
            <action name = "log unexpected client message" />
            <action name = "terminate" />
        </event>
    </state>

### Client and Server Properties

In your server code, you have two structures, client_t and server_t. Note that the client_t structure MUST always start with these variables (the message uses whatever protocol name you defined):

    server_t *server;           //  Reference to parent server
    hello_msg_t *message;       //  Message from/to clients

And the server_t structure MUST always start with these variables:

    zsock_t *pipe;              //  Actor pipe back to caller
    zconfig_t *config;          //  Current loaded configuration

### Client Connection Expiry

If you define an "expired" event anywhere in your dialog, the server will automatically expire idle clients after a timeout, which defaults to 60 seconds. It's smart to put this into a superstate:

    <state name = "defaults">
        <event name = "PING">
            <action name = "send" message = "PONG" />
        </event>
        <event name = "expired">
            <action name = "terminate" />
        </event>
    </state>

To tune the expiry time, use this method (e.g. to set to 5 second):

    hello_server_set (self, "server/timeout", "5000");

The server timeout can also come from a configuration file, see below. It is good practice to do heartbeating by sending a PING from the client and responding to that with a PONG or suchlike. Do not heartbeat from the server to clients; that is fragile.

### Server Configuration File

You can call the 'configure' method on the server object to configure it, and you can also call the 'set' method later to change individual configuration options. The configuration file format is ZPL (ZeroMQ RFC 5), which looks like this:

    #   Default zbroker configuration
    #
    hello_server
        echo = I: starting hello service on tcp://*:8888
        bind
            endpoint = tcp://*:8888

    #   Apply to all servers that load this config file
    server
        timeout = 10        #   Client connection timeout
        background = 1      #   Run as background process
        workdir = .         #   Working directory for daemon

'echo' and 'bind' in the 'hello_server' section are executed automatically.

### CZMQ Reactor Integration

The generated engine offers zloop integration so you can monitor your own sockets for activity and execute callbacks when messages arrive on them. Use this API method:

    //  Poll socket for activity, invoke handler on any received message.
    //  Handler must be a CZMQ zloop_fn function; receives server as arg.
    static void
    engine_handle_socket (server_t *server, void *socket, zloop_fn handler);

The engine invokes the handler with the 'server' as the argument. Here is the general style of using such a handler. First, in the 'server_initialize' function:

    engine_handle_socket (self, self->some_socket, some_handler);

Where 'some_socket' is a ZeroMQ socket, and where 'some_handler' looks like this:

    static int
    some_handler (zloop_t *loop, zmq_pollitem_t *item, void *argument)
    {
        server_t *self = (server_t *) argument;
        zmsg_t *msg = zmsg_recv (self->some_socket);
        if (!msg)
            return 0;               //  Interrupted; do nothing
        zmsg_dump (msg);            //  Nice during development
        ... process the message
        return 0;                   //  0 = continue, -1 = end reactor
    }

Similarly you can tell the engine to call a 'monitor' function at some specific interval, e.g. once per second. Use this API method:

    //  Register monitor function that will be called at regular intervals
    //  by the server engine
    static void
    engine_set_monitor (server_t *server, size_t interval, zloop_timer_fn monitor);

Call this in the 'server_initialize' function:

    engine_set_monitor (self, 1000, some_monitor);

Were 'some_monitor' looks like this:

    static int
    some_monitor (zloop_t *loop, int timer_id, void *argument)
    {
        server_t *self = (server_t *) argument;
        ... do regular server maintenance
        return 0;                   //  0 = continue, -1 = end reactor
    }

## The Client Generator

The zproto project lets you generate full asynchronous client stacks in C to talk to the server engines. Overall the model and toolchain is similar to that used for servers. See the zproto_client_c.gsl code generator. The main differences is that:

* A client manages one connection to a server;
* We generate a full CLASS API that wraps the client actor with conventional methods;
* The client XML model has a language for defining these methods.

### The zproto Client Model

Your input to the code generator is two XML files that defines a set of 'states', and the protocol messages as used to generate the codec. Here is a minimal 'hello_client.xml' example that defines a Hello, World client:

    <class
        name = "hello_client"
        title = "Hello Client"
        script = "zproto_client_c"
        protocol_class = "hello_msg"
        project_header = "czmq.h"
        package_dir = "."
        >
        <state name = "start">
            <event name = "constructor" next = "connected">
                <action name = "connect to server" />
                <action name = "send" message = "HELLO" />
            </event>
        </state>
        <state name = "connected">
            <event name = "WORLD" next = "connected">
                <action name = "terminate" />
            </event>
        </state>
        <method name = "constructor">
        Connect to server.
            <field name = "endpoint" type = "string" />
        </method>
    </class>

The zproto_client_c.gsl code generator produces:

* A .h file that acts as the public API for your client
* An .inc file called an *engine* that runs the state machine
* The first time, a skeleton .c file for your client class

The client is a "actor" built on the CZMQ/zactor class. The generated zactor accepts these messages:

    VERBOSE
    $TERM

Plus one message for each method defined in the model, including the pre-defined "constructor" and "destructor" methods (called after, and before construction and destruction respectively).

The client stack is then wrapped in a classic CLASS API like this:

    //  Create a new hello_client
    hello_client_t *
        hello_client_new (const char *endpoint);

    //  Destroy the hello_client
    void
        hello_client_destroy (hello_client_t **self_p);

    //  Enable verbose logging of client activity
    void
        hello_client_verbose (hello_client_t *self);

    //  Return actor for low-level command control and polling
    zactor_t *
        hello_client_actor (hello_client_t *self);

Names are case insensitive. By convention however we use uppercase for protocol events and methods. Protocol events can also contain embedded spaces or hyphens, which are mapped to underscores. In this case, HELLO and WORLD are two messages that must be defined in the hello_msg.xml file.

To build this, do:

    gsl -q -trace:1 hello_client.xml

The first time you do this, you'll get a hello_client.c source file. You can edit that; it won't be regenerated. The generated code goes, instead, into hello_client_engine.inc and hello_client.h. Take a look if you like.

The trace option shows all protocol messages received and sent.

There are two predefined actions: "send", which sends a specific protocol message, and "terminate", which ends the client. Other actions map to functions in your own code, e.g.:

    <state name = "start">
        <event name = "HELLO" next = "start">
            <action name = "tell the user hello too" />
            <action name = "send" message = "WORLD" />
        </event>
    </state>

    ...

    static void
    tell_the_user_hello_too (client_t *self)
    {
        puts ("Hello, World!");
    }

Your client code (the actions) gets a small API to work with:

    //  Set the next event, needed in at least one action in an internal
    //  state; otherwise the state machine will wait for a message on the
    //  dealer socket and treat that as the event.
    static void
    engine_set_next_event (client_t *client, event_t event);

    //  Raise an exception with 'event', halting any actions in progress.
    //  Continues execution of actions defined for the exception event.
    static void
    engine_set_exception (client_t *client, event_t event);

    //  Set wakeup alarm after 'delay' msecs. The next state should
    //  handle the wakeup event. The alarm is cancelled on any other
    //  event.
    static void
    engine_set_wakeup_event (client_t *self, size_t delay, event_t event);

    //  Set a heartbeat timer. The interval is in msecs and must be
    //  non-zero. The state machine must handle the "heartbeat" event.
    //  The heartbeat happens every interval no matter what traffic the
    //  client is sending or receiving.
    static void
    engine_set_heartbeat (client_t *self, size_t heartbeat);

    //  Set expiry timer. Setting a non-zero expiry causes the state machine
    //  to receive an "expired" event if is no incoming traffic for that many
    //  milliseconds. This cycles over and over until/unless the code sets a
    //  zero expiry. The state machine must handle the "expired" event.
    static void
    engine_set_expiry (client_t *client, size_t expiry);

    //  Set connected to true/false. The client must call this if it wants
    //  to provide the API with the connected status.
    static void
    engine_set_connected (client_t *client, bool connected);

    //  Poll socket for activity, invoke handler on any received message.
    //  Handler must be a CZMQ zloop_fn function; receives server as arg.
    static void
    engine_handle_socket (client_t *client, zsock_t *socket, zloop_reader_fn handler);

### Heartbeating

Use the engine_set_heartbeat method to generate a regular "heartbeat" event when connected, and send a PING each time. The server needs to respond with a PONG. Then, set an expiry timeout of 2 or 3 times the heartbeat interval, and use this to detect a dead server.

### Superstates

Superstates are a shorthand to reduce the amount of error-prone repetition in a state machine. Here is the same state machine using a superstate:

    <state name = "subscribing" inherit = "defaults">
        <event name = "SUBSCRIBE OK" next = "connected">
            <action name = "signal success" />
        </event>
    </state>

    <state name = "disconnecting" inherit = "defaults">
        <event name = "GOODBYE OK">
            <action name = "terminate" />
        </event>
        <event name = "expired">
            <action name = "terminate" />
        </event>
    </state>

   <state name = "defaults">
        <!-- Server didn't respond for some time -->
        <event name = "expired" next = "reconnecting">
            <action name = "use connect timeout" />
            <action name = "send" message = "HELLO" />
        </event>
        <!-- Server lost our connection state -->
        <event name = "INVALID" next = "reconnecting">
            <action name = "use connect timeout" />
            <action name = "send" message = "HELLO" />
        </event>
        <event name = "*">
            <!-- Discard any other incoming events -->
        </event>
    </state>

For complex protocols you can collect error handling together using the wildcard event, "*", which means "all other protocol events in this state". For example:

    <state name = "defaults">
        <event name = "PING">
            <action name = "send" message = "PONG" />
        </event>
        <event name = "*">
            <action name = "log unexpected server message" />
            <action name = "terminate" />
        </event>
    </state>

### Before/After State Actions

As another way to reduce error-prone repetition, it is possible to add actions to be executed for any event that transitions to or from a given state. This is modelled with a before or after element containing one or more action elements inside of a state element. The given actions will be executed only when the state of the machine changes to or from that state (due to an event that has the "next" attribute defined).

    <state name = "active" inherit = "defaults">
        <before>
            <action name = "start underlying service" />
        </before>
        <event name = "REQUEST">
            <action name = "store metadata" />
            <action name = "forward to underlying service" />
        </event>
        <event name = "reply from underlying service">
            <action name = "send" message = "REPLY" />
            <action name = "clear metadata" />
        </event>
        <after>
            <action name = "stop underlying service" />
        </after>
    </state>

### Client Properties

In your client code, you have a client_t structure. Note that the client_t structure MUST always start with these variables (the msgout and msgin will use whatever protocol name you defined):

    zsock_t *pipe;              //  Actor pipe back to caller
    zsock_t *dealer;            //  Socket to talk to server
    my_msg_t *msgout;           //  Message to send to server
    my_msg_t *msgin;            //  Message received from server

### Client Expiry Timer

If you define an "expired" event anywhere in your dialog, the client will automatically execute an expired_event after a timeout. To define the expiry timeout, use engine_set_expiry (). The expired event will repeat whenever there is no activity from the server, until you set a expiry of zero (which ends it).

### Method Framework

To simplify the delivery of a conventional non-actor API, you can define methods in your state machine. Here are some examples taken from real projects:

    <method name = "constructor" return = "status">
    Connect to server endpoint, with specified timeout in msecs (zero means
    wait forever). Constructor succeeds if connection is successful.
        <field name = "endpoint" type = "string" />
        <field name = "timeout" type = "number" />
        <accept reply = "SUCCESS" />
        <accept reply = "FAILURE" />
    </method>

    <method name = "subscribe" return = "status">
    Subscribe to all messages sent to matching addresses. The expression is a
    regular expression using the CZMQ zrex syntax. The most useful elements
    are: ^ and $ to match the start and end, . to match any character,
    \s and \S to match whitespace and non-whitespace, \d and \D to match a
    digit and non-digit, \a and \A to match alphabetic and non-alphabetic,
    \w and \W to match alphanumeric and non-alphanumeric, + for one or more
    repetitions, * for zero or more repetitions, and ( ) to create groups.
    Returns 0 if subscription was successful, else -1.
        <field name = "expression" type = "string" />
        <accept reply = "SUCCESS" />
        <accept reply = "FAILURE" />
    </method>

    <method name = "publish">
    Publish a message on the server, using a logical address. All subscribers
    to that address will receive a copy of the message. The server does not
    store messages. If a message is published before subscribers arrive, they
    will miss it. Currently only supports string contents. Does not return a
    status value; publish commands are asynchronous and unconfirmed.
        <field name = "address" type = "string" />
        <field name = "content" type = "string" />
    </method>

    <method name = "recv" return = "content" immediate = "1" virtual ="1">
    Receive next message from server. Returns the message content, as a string,
    if any. The caller should not modify or free this string. This method is
    defined as "immediate" and so does not send any message to the client actor.
    Virtual 1 means recv method returns msg part of incomming message. Virtual 0 means
    recv returns 0/-1 and message parts are accessible using client methods.
        <accept reply = "MESSAGE" />
    </method>

    <!-- These are the replies from the actor to the API -->
    <reply name = "SUCCESS">
        <field name = "status" type = "number" />
    </reply>

    <reply name = "FAILURE">
        <field name = "status" type = "number" />
        <field name = "reason" type = "string" />
    </reply>

    <reply name = "MESSAGE">
        <field name = "sender" type = "string" />
        <field name = "address" type = "string" />
        <field name = "content" type = "string" />
    </reply>

Each method is implemented as a classic CLASS method, with the public API in the generated .h header, and the body in the generated .inc engine. For example:

    //  ------------------------------------------------------------------
    //  Subscribe to all messages sent to matching addresses...

    int
    mlm_client_subscribe (mlm_client_t *self, const char *stream, const char *pattern)
    {
        assert (self);
        zsock_send (self->actor, "ss", "SUBSCRIBE", stream, pattern);
        if (s_accept_reply (self, "SUCCESS", "FAILURE", NULL))
            return -1;              //  Interrupted or timed-out
        return 0;
    }

When the calling application uses the method, this can do any or several of these things:

* Send a message to the client actor, if the method does not have the 'immediate = "1"' attribute. This generates an event in the client state machine, corresponding to the method name.

* Wait for one of a set of replies from the actor, and store reply properties in the client object. These are defined by one or more <accept> tags.

* Return a propery to the caller. This is defined by the "return" attribute of the <method> tag.

All possible replies are defined as <reply> objects. The actor's replies are always several frames. The first is the reply name, and the following are the reply fields.

We currently support only two field types: string, and number, which map to char * and int.

The fields you pass in a method are accessible to client state machine actions via the self->args structure. For example:

    if (zsock_connect (self->dealer, "%s", self->args->endpoint)) {
        engine_set_exception (self, error_event);
        zsys_warning ("could not connect to %s", self->args->endpoint);
    }

### Custom hand-written methods

Within your state model you can include another XML file with custom hand-written methods to easily extend the state maschine. To do so include the following:

    <custom filename = "hello_client_custom.xml" language = "C" />

The hello_client_custom.xml can contain three sections `<header>`, `<source>` and `<api>`. The contents of the `<header>` and `<source>` section will be placed into the hello_client.(h|c) files. The `<api>` defines a zproject API for your custom code and can only be used in conjunction with zproject. Please note that `<header>` and `<api>` are exclusive as the header will be generated from the API by zproject. The hello_client_custom.xml may look like:

    <header>
        //  Print the attributes of this class
        void
            hello_client_print (hello_client_t *self)
    </header>
    <source>
        //  -------------------------------------------------------------------
        //  Print the attributes of this class

        void
        hello_client_print (hello_client_t *self)
        {
            printf ("hello_client {\n");
            printf ("  last command %s\n", self->command);
            ...
            printf ("}\n");
        }
    </source>

When using zprojects API you'll need to escape '<' and '>' for now:

    <api>
        &lt;method name = "print"&gt;
            Print the attributes of this class
        &lt;/method&gt;
    </api>
    <source>
        //  -------------------------------------------------------------------
        //  Print the attributes of this class

        void
        hello_client_print (client_t *client)
        {
            if (client) {
                s_client_t *self = (s_client_t *) client;
                printf ("hello_client {
                printf ("  heartbeat %lu\n", self->heartbeat);
                printf ("  current event %lu\n", self->event);
                printf ("  current state %lu\n", self->state);
                ...
                printf ("}\n");
            }
        }
    </source>

## Protocol Design Notes

This section covers some learned experience designing protocols, using zproto and more generally:

### Heartbeating and Client Expiry

The simplest and most robust heartbeat / connection expiry model appears to be the following:

* The server disconnects unresponsive clients after some expiry timeout, which you can set using the SET message and the "server/timeout" property. A good expiry timeout is perhaps 3 to 10 seconds. The minimum allowed is 2000 milliseconds.

* The client heartbeats the server by sending a PING heartbeat every second if there is no other activity. You can do this by calling "engine_set_timeout (self, 1000);" in the client and in expired_event handling, send a PING command (or similar).

* The server responds to PING commands with a PING-OK (or similar), when it is in a valid connected state. When the server does not consider the client as connected, it responds with INVALID (or similar).

* The client accepts and discards PING-OK. If it receives INVALID, it re-starts the protocol session by sending OPEN (or similar).

* The server accepts OPEN in all external states and always treats this as a request to start a new protocol session.

This approach resolves stale TCP connections, as well as dead clients and dead servers. It makes the heartbeating interval a client-side decision, and client expiry a server-side decision (this seems best in both cases).

## For More Information

Though [the Libero documentation](http://legacy.imatix.com/html/libero/) is quite old now, it's useful as a guide to what's possible with state machines. The Libero model added superstates, substates, and other useful ways to manage larger state machines.

The current working example of the zproto server generator is the [zeromq/zbroker](https://github.com/zeromq/zbroker) project, and specifically the zpipes_server class.

You can find [GSL on Github](https://github.com/zeromq/gsl) and there's a [old backgrounder](http://download.imatix.com/mop/) for the so-called "model oriented programming" we used at iMatix.

## This document

_This documentation was generated from zproto/README.txt using [Gitdown](https://github.com/zeromq/gitdown)_
