# Composing POETS messages

In POETS, a _message_ travels along a graph edge from an output port of a vertex, where
it is produced by a _send handler_, to an input port of a vertex, where it is consumed
by a _receive handler_. When a message is consumed, the receive handler updates the state
of the receiving port and vertex.

Messages are produced, delivered and consumed asynchronously, hence one may need to
introduce _message queues_ (also referred to as _buffers_ in hardware) as a temporary
storage for messages as they travel along the interconnect. These queues may be costly
both when implementing POETS engines in hardware and when simulating them in software.

In this memo we discuss how one can remove or reduce the size of the queues when messages
can be composed in an _effect-preserving_ way, or using standard mathematical terminology,
when messages form a _monoid action_ with respect to the receive handlers and port/vertex
states.

**NB:** Message queues may be necessary when implementing high-performance interconnect
in order to reduce critical request-acknowledgement cycles between senders and receivers.
It is therefore not clear at this stage whether any hardware savings will be possible
with this technique. A further investigation/benchmarking is required. An alternative is
to apply the optimisation at the software level when receiving messages and dispatching them
to individual handlers.

## Message action and composition

Let _M_ be a set of messages, and _S<sub>p</sub>_ and _S<sub>v</sub>_ be sets of
port and vertex states, respectively. We introduce the operator ▷ as a concise
mathematical notation for describing the _action_ of a message on a receiving port and
vertex. Given a message _m_ ∈ _M_ and a state _s_ ∈ _S<sub>p</sub>_ × _S<sub>v</sub>_,
we write _m_ ▷ _s_ to denote the new state of the port and vertex after receiving the 
message. In other words, _m_ ▷ _s_ denotes the action of message _m_ on state _s_.

Using this notation, we can express the effect of two messages _a_ and _b_ on the receiving
vertex as follows: _a_ ▷ _b_ ▷ _s_ (for convenience ▷ associates to the right). Note that 
message _b_ arrives and acts first and messages do not necessarily commute, i.e. _a_ ▷ _b_ ▷ _s_
and _b_ ▷ _a_ ▷ _s_ are not neccessarily the same.
The combined action of a queue containing _k_ messages arrived in the order
_m<sub>1</sub>_..._m<sub>k</sub>_ on state _s_ can therefore be denoted as
_m<sub>k</sub>_ ▷ ... ▷ _m<sub>2</sub>_ ▷ _m<sub>1</sub>_ ▷ _s_.

Suppose that messages can be composed using the operator ◇ : M × M → M in such a way that
_a_ ▷ _b_ ▷ _s_ = (_a_ ◇ _b_) ▷ _s_.
In other words, instead of calling the receive handler twice on messages _a_ and _b_, one can
instead first compose the messages into _m_ = _a_ ◇ _b_ and then call the receive handler on
the resulting message _m_ only. This allows to replace a message queue with a single message
slot and a message composition function.

Indeed, let _e_ ∈ _M_ stand for an _empty message_ that acts on the state as the identity
transformation: _e_ ▷ _s_ = _s_.
Then the receiving slot can be initialised to contain _e_ and whenever a new message _m_ arrives
it can be composed with the current slot contents _c_ using the ◇ operator: _c<sup>new</sup>_ = _m_ ◇ _c_.
Whenever a receive handler is ready to consume the message, it removes _c_ from the slot
replacing it with the empty message _e_, and uses _c_ to act on the current state: _c_ ▷ _s_.

The structure described above appears in many applications and has been studied under the name
of [monoid action](https://en.wikipedia.org/wiki/Semigroup_action), which is explained by the
fact that a set of messages _M_ is a _monoid_ whenever it has an identity element _e_ ∈ _M_ and
an associative composition operator ◇ : M × M → M such that _e_ ◇ _m_ = _m_ ◇ _e_ = _m_. 

## Examples

Two simple examples are:
* In some applications the combined action of two messages is equivalent to the action of the 
_fresher_ one, so older messages can be discarded. This can be implemented by defining a
composition operator ◇ that compares the time stamps of two messages and returns the one whose
time stamp is more recent.
* If messages are aggregated by the receive handler then the aggregation function can be
incorporated into the composition operator ◇ itself. For example, if the receive handler
computes the sum of values in incoming messages, one can simply let ◇ to be the addition.
