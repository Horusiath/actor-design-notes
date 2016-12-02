# Transport protocols

In order to communicate, two processes needs to specify some kind of communication protocol between them. However in greater detail we can define various different types of transport here:

- Remote - so how two processes will emit messages over the network. This however can also vary (i.e. machines in the same datacenter vs mobile client)
- Persistence - so how the messages/events will be stored in a persistent storage.

Second option matters when we'll start to talk about serialization.

It's clear that sending messages between processes and serialization are thighly coupled concepts:

- When we want to send a message between two machines in the same data center, we may decide to optimize for throughtput.
- When we want to send messages between cluster and mobile client, other things starts to matter: message size over serialization time, security over anything else.
- When we want to persist everlasting events (see: *eventsourcing* patterns) we need to take into account, what database backend do we target, but also what version tolerance guarantees we want to assure (as event type schema may change over time). NOTE: maybe for this case serialization is not correct keyword and may be a bit missleading.

For those reasons, there shouldn't be a single global "SerializationManager" class specific for cluster node, but rather something like `SerializationProfile` for any two endpoints (there should be a `default` profile however, common for each cluster node).

## Sessions

Session denotes a continuous connection between two existing processes. This doesn't need to be equal to TCP connection - we can have two processes, which lost connection with each other, some time later a new connection has been established, but we still have the same 2 processes.

Why we could use transport sessions for? This again goes back to a problem of serialization. The most performant serializers are usually schema-based: we need to ensure at compile time that, two communicating endpoints will share the same view over messages schemas. This may be hard. 

Another approach is to encode message schema into payload itself - all JSON/BSON formats work this way.

However if we i.e. establish session, we could make two endpoints dynamically share/upgrade their message schemas and map it to some schema id. Later, when the messages of that type are shared, instead of encoding schema into payload, we could just use an id. From other perspective, we could create a protocol for schema negotiation and upgrades, unlike in the compiled schema approach. 