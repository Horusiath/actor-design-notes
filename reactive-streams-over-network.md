# Reactive streams over the network

In case of reactive streams a well known behavior is demand-driven dynamic push-pull model:

- Consumer sends a demand value
- Producer pushes events and decrements a demand with each value. Producer won't send more events than the demand provided.

## Problem with standard reactive streams protocol over network

Where is the problem? Network is unreliable - that means, when event is send over the network, you have no guarantee, that it will be received on the consumer end. So what's the proposition?

Under normal conditions we could go with pull model - consumer sends demand(1) to producer, so basically each event must be acknowledged. But I think, we can do better than that. How?

We know that we are able to provide guaranteed order delivery between two endpoints (this is a property of Erlang and Akka networking layer). So what we can to is to attach a control number to each message. Number must me monotonically increasing (by 1) between two given endpoints. This way consumer is able to detect missed messages (if previous event has controlNr = N, we can assert if next event will have controlNr = N+1). In that case we can decide about the stream behavior:

- Should we notify missing message using `OnError`? 
- Should we try to redeliver a message?

### Batching messages over network

Other nice property of variable demand is that since we know the demand, we are able to send a batch of messages at once if only consumer is able to provide them. This can be easily achieved via `Conflate` stage - it would be probably a good idea to extend conflate with windowing (so when new demand arrives, we don't send a whole conflated batch in one go, but cut it into pieces and send them instead).