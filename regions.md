# Regions

### Motivation

Currently, there are two major fronts on how to compose and work with actor systems.

Old approach (Erlang, Akka) defines actors as entities spawned on specific machine, with a fixed lifetime cycle. This comes with several traits:

- Actors needs to be created before calling them.
- Actor references (PIDs / `ActorRef`s) can point to dead or non existing actors - causing necessity of having dead letters and dead watch mechanics.
- Actor addresses are locally unique, making a hard to locate actor in a global (cluster-wide) scope, as we need to determine on which node it lives.
- Each actor can have it's own mailbox, giving a lot space for optimizations (statically typed mailboxes, multi-write/single-read implementations etc).
- Communication via actor references is very fast: we don't need to validate actor's location each time, also internal implementation may point directly to correlated mailbox.

There are more pros/cons, but I won't cover them here. A lot of those issues can be mitigated (see riak_core or akka.cluster.sharding).

A new approach called virtual actors (Orleans, Orbit) defines actors as everliving entities that can be spawned anywhere in the cluster. Some the their traits are:

- Actor incarnations/activations are created ad-hoc anywhere on the cluster, when necessary.
- Actors are identified by global keys. Location of keys themselves is kept using distributed hashing tables (DHT) - this also means that we need to provide additional ways on how to invalidate multiple activations of the same actor between nodes and resolve all conflicts that this duality comes with.
- As actors can be rebalanced between nodes at any time, every time we contact with them we need to validate their location, which is slow.
- Since actors are not bound together, there is a risk of non-optimal remote communication - causing a necessity of having different placement strategies, pinned actors or special algorithms (see ActOp).
- Actors creation may be a lot heavier, making them to be useless in some scenarios.
- In order to be able to migrate actor mailboxes efficiently, they had to be kept separate from actor instances themselves.

Both approaches have good and bad sides. Their specific downfalls is causing a lot of mechanics to be implemented.

## Idea

Idea here is to group actors into so-called regions. Regions are containers for actors, which keep the promise of maintaining locality boundary between actors - actors living inside the same region are guaranteed to always live in the same process. However unlike partitions/shards, they are not specific to actor type - a single region can contain a different types of actors and collocation of actors living on the same region is explicit - not automatic like in case of partition resolution. What regions could do?:

- Keep a list of all actors linked together.
- In order to keep rebalancing easy to track without need of special techinques (like passivation), each region should have a mailbox bound to it. 
- 

Question how to obtain an actor reference, and still be able to keep which region it belongs to?