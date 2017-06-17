# Reactive streams all the way down

It turned out that streams are pretty universal primitive. They provide universal units of composition (`IPublisher<>`, `ISubscriber<>` and `IProcessor<,>`), that could be potentially reused. There are two places that could be a great point for graph-based architecture: remote and persistence (eventsourcing) protocols.

## Streams-based protocols for the cluster


    +--------------------------+
    | Aeron | TCP | Websockets |
    +--------------------------+
    |         RSocket          |
    +--------------------------+
    |         Remoting         |
    +--------------------------+
    |  Cluster  |  Replication |
    +-----------+--------------+
    |           |     CRDT     |
    +-----------+--------------+

### Socket protocol 

First point where reactive streams design could be already applied is transport protocol. This is heavily related to the next layer (reactive sockets), which sets certain constraints on the transport: this is why 3 major propositions are TCP, Websockets or Aeron. For TCP, we could adapt one of the existing frameworks (DotNetty, System.IO.Pipelines) or extend power of actor based frameworks over sockets (i.e. what Akka does with socket actors composed inside streams).

### Reactive sockets

At this point once we have transport layer we can now abstract itself as a reactive stream, giving us desired semantics. From that point we no longer need to rely on the fact, that we have a network to deal with.

### Remoting

This is first big layer as at this point we need to abstract several things:

1. Hide actor address resolution.
2. Provide session-aware serialization. Since we exactly know what data we've already send through the connection, we can cache it in the form of compression tables and send only compresed data IDs.
3. Provide connection maintenance service. This covers connection reinitialization, quarantine and death pacts.

Remoting layer basically transforms potentially killable stream of byte buffers into infinite stream of objects.

### Cluster

On top of remoting engine we can now provide other semantics, i.e. implementation of cluster membership protocols (i.e. [SWIM](http://www.cs.cornell.edu/~asdas/research/dsn02-SWIM.pdf) protocol).

### Replication

Other big part is a replication layer. This allows us to broadcast data over the network in organized and reliable fashion. A minimal approach is to have a replication network definition: it's basically a directed graph, which tells what data (filtering) should be carried where (routing). 

More advanced approach would be to provide algorithms for building replication networks (i.e. spanning trees, see [Plumtree](https://github.com/helium/plumtree)).

Replication network should also cover reliable casual delivery for sake of CRDTs and exactly-once delivery semantics.

### CRDT

On top of Replication layer we now have everything what's needed to build a conflict-free replicated data types, giving us the abilities to construct eventually consistent databases on top of existing cluster. CRDTs could be also used by the cluster protocol itself for constructing membership procotols, but this is probably an overkill. 

CRDTs could also combine replication streams with eventsourced streams for persistence.

## Stream-based eventsourcing

    +------------------------------+
    |      Eventsourced actor      |
    +------------------------------+
    | Read adapter ~ Write adapter |
    +------------------------------+
    | Read adapter ~ Write adapter |
    +------------------------------+
    |          Event log           |
    +------------------------------+

You can thing of event logs as if they were simple `IProcessor<,>` stages.

```csharp
type EventLog = IProcessor<EventReq, EventRep>
```

Since event logs require evolution strategy for their events, we could combine other stages to transform data back and fort between requestors (actors) and event log.

Expressing event logs as reactive streams give us the power to greatly optimize them at low cost. For example the whole batching layer could be simply expressed as single `Conflate` stage.