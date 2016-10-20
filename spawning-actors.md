# Spawning actors 

Actors could be spawned in one of two ways:

### Spawn

`IChannelRef<T> Spawn<T>(Receive<T> behavior, string name)` spawns actor somewhere in the cluster. The exact location doesn't matter here. Actors can be addressed by using their name, which needs to be globally unique.

Actors location could be stored using distrinbuted hash tables and potentially rebalanced using ActOp algorithm (however think if it's possible to abstract that due to ability to use linked actors).

### Spawn linked actors

`IChannelRef<T> SpawnLinked<T>(Receive<T> behavior, string name)` spawns an actor in the exact same location where a spawning actor resides. Moreover those two actors share two things:
    - Affinity: they are guaranteed to **always** work on the same node. If parent is going to be rebalanced somewhere else, all of it's linked actors will rebalance with him.
    - Live span: linked actors will never outlive their parent.

Since linked actors provide islands of locality, we have possibility to optimize code in a ways that are not possible if actors could be rebalanced onto different machines at random time.

This way we could group linked actors into structures (let's call them `Region`s). It would make easier to work with regions than single actors in terms of rebalancing - once we need to rebalance part of the actors on the node, we could count balancing cost for each region in order to calculate the best fit part of the cluster.

![regions](./images/cluster-node-and-regions.png)

Other thing regions could potentially do is to count the communication frequency of the actors that are not potentially linked, as even thou the direct "link" relationship doesn't existis, in case of rebalancing it would be better to rebalance most communicating actors together on the same node. Linking is also useful optimization here, as we don't need to track communication frequency between every possible pair of actors (as linked actors will be migrated together anyway), but only between regions. This would be optimization over existing ActOp algorithm. 

![region communication frequency](./images/region-communication-freq.png)

Actors could also cooperate and inform each other about their changes.

> In Akka we can watch over actor and be informed about actor's termination. However in case of `Spawn` we should also be somehow notified about things like rebalancing. Moreover we should be able to filter, if we are interested about rebalancing or termination, to reduce the footprint of the message exchange that we don't need.