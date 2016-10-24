# Working with the cluster from the outside

## Service description

Sometimes there may be a need to work with the cluster from outside of it. From internal point of view it could look like that: any actor interested in serving data from the outside should register itself in some kind of `ClusterReceptionist`, saying by what address can it be accessed and what contract does it fulfill. All of those contracts should be aggregated and exposed to external users in form of the protocol descriptor - a data structure describing valid contracts exposed by the cluster. gRPC could be a great inspiration here:

- Services should cover more than just request-response message pattern. Beside it, possible patterns could be:
    - Reactive stream of inputs / single response.
    - Single request / reactive stream of outputs.
    - Combination of both above - reactive stream working in duplex mode.
- Dealines - ability to tell services inside of the cluster, that the request has timed out, and order them to cancel working on that operation or just discard it if it lays in mailbox.
- Idea: would it be feasible to describe a compensating actions as part of the protocol? They don't necessarily have to be implemented by each contract, but once task has failed or cancelled due to deadline, it might be useful to (semi)automatically apply compensating actions right away?

By giving a cluster service descriptor, we could give consumers a great tooling capabilities, even ability to generate typesafe cluster clients on demand without need of sharing some contract libraries. This would be especially useful, if cluster protocol was described in form of a spec and could work across many languages. This would also require to describe a common type system, to make sure, that any of the communicating parties are able to always parse the response.  