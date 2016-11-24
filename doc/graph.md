# POETS graph

At the abstract level POETS graphs contain _vertices_ connected by _edges_. Edges are
directed and connect an _output port_ of a vertex to an _input port_ of a vertex. Edges
are used to communicate _messages_ between ports. Messages can have different types.

A graph is _well-formed_ if the following conditions are met:

* Every input port belongs to exactly one vertex.
* Every output port belongs to exactly one vertex.
* Every edge connects an output port and an input port of matching message types.

## Vertex state and events

Each vertex has internal _state_ that can be changed atomically as a result of an _event_ occurrence.
There are three types of events: _receive_, _compute_ and _send_ events.

In addition to vertex state, each vertex is characterised by a few constant parameters, such as
the vertex identification number and/or the vertex address, which we refer to as _properties_.

## Event handlers

An event occurrence can be described by a pure _handler_ function, whose type differs depending
on the type of the event. 

* Receive event handlers have type _Receive : S × P × M → S_, that is: given a vertex state, its properties
and an incoming event, the handler computes the next state of the vertex.

* Compute event handlers have type _Compute : S × P → S_, that is: given a vertex state and its properties,
the handler computes the next state of the vertex.

* Send event handlers have type _Send : S × P → S × Maybe M_, that is: given a vertex state and its properties,
the handler computes the next state of the vertex and _may_ also return a message to be sent to another vertex.

Note, a vertex can have several receive and send event handlers, one for each input and output port, respectively.

## Vertex readiness to compute and send

We typically assume that a vertex is always ready to receive an incoming message and the corresponding event
handler can be invoked whenever a new message has arrived.

The _readiness_ of compute and send handlers can be determined from the vertex state and properties using the
following auxiliary functions: 

* Ready-to-compute _rtc : S × P → Bool_ returns _True_ if the vertex requires computation resources to evolve its
current state (e.g. to compute the payload for a new message to be sent). The compute handler should be called
only when ready-to-compute is _True_.

* Ready-to-send _rts : S × P → Bool_ returns _True_ to indicate that the vertex is ready to send a message at the
earliest opportunity. The send handler should only be called when ready-to-send is _True_.

Note: since a vertex can have multiple send handlers (one per output port), we can consider the following variants
of the ready-to-send function: _rts : S × P × O → Bool_ (where _O_ is the output port) and _rts : S × P → Maybe O_.
The latter either returns _Nothing_ if the vertex is not ready to send, or a specific port that is ready to send.

## Low-level event handlers

In the interest of efficiency, the actual low-level event handlers written in C will not be implemented as pure functions
but instead will be mutating vertex states and messages in place. Below we list prototypes of C functions that will act
as low-level handlers.

* Receive handler `void Receive(const M *message)` receives a pointer to a constant message, mutates the vertex state,
and can read its properties.
* Compute handler `void Compute()` performs the computation, mutates the vertex state, and can read its properties.
* Send handler `bool Send(M *message)` constructs a message to be sent at the given location (specified by the pointer).
It can also mutate the vertex state and read its properties. The handler returns _True_ to indicate that the message has
been constructed or _False_ if the sending has been cancelled and no message should be sent.
