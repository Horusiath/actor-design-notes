# Cluster

Cluster state exchange could be done with CRDT data structures.

Ideally cluster membership management should be able to be separated (so it can be used as a library by programmers who want cluster capabilities without the whole burden of this particular actor framework).

## Cluster initialization

## Cluster partition resolver strategy

- Keep to reference node
- Keep to the oldest known node
- Keep majority of nodes
- Call third party plugin (i.e. Azure Service Fabric, Consul, Zookeeper, etcd) to resolve the cluster node outcome.

## Event based persistence

### Replication networks

### Operation-based Conflict-Free Replicated Data Types

### Read-Atomic Multi-Partition transactions