# Channels

Channels are meant to provide means of exchanging messages, abstracting things such as transport layer and buffering.

- Channels should provide an asynchronous way of message exchange (this way they differ from CSP channels). This is necessary if we are going to use them for message transport in distributed scenarios.
- Channels should abstract the underlying transport layer - this is a key to location transparency concept.
- Channel references should be serializable. It should be possible to include channel refs inside messages.

Last two options are more directed to concepts of Akka's `IActorRef`s. Moreover there's also one important note here - **channels should be composable**. There are many different kinds of composition:

- We can look at priority channels as a composition of X different channels. This would be useful for signals mechanism of actors (so when we have actors that should receive not only user messages but also system signals).
- Sometimes we don't want to do any kind of advanced logic, i.e. simply redirect message to another channel or broadcast it. This would also require channels to expose some kind of `CanSend` / `CanReceive` property to check if other channel (possibly behind remoting barrier) is listening.

## Types of channels

### Shared interfaces

```csharp
interface IReceiveChannel<T> 
{
    // Check if channel is open to read. This may be usefull for network channels, which may not be 
    // readable when underlying transport is closed.
    bool CanReceive { get; }

    // Try to read from channel.
    bool TryReceive(out T message);

    // Create a task that will complete once channel will receive some message.
    Task<T> ReceiveAsync();
}

interface ISendChannel<T> 
{
    // This can be basically Int64 value used to uniquelly identify channel among others in a cluster
    Uid Address { get; }

    // Check if channel is open to write. This may be false, once channel has been disposed,
    // closed (i.e. IVar), underlying transport (network channel) is closed or the message buffer has been reached.
    bool CanSend { get; }

    // Try to write to channel if possible.
    bool TrySend(T message);
}

interface IChannel<T> : IReceiveChannel<T>, ISendChannel<T> 
{
}
```

One of the composition methods could be message redirection. The most simplistic case: we have a local actor with channel. At some point this actor has been moved to remote node. Instead of invalidating all channel refs, we could simply link/redirect existing one to a new network channel between local and remote node. 

### IVar channel

Most simplistic write-once channel. It automatically closes (`CanSend = false`) when value has been written to it. Useful for on-demand channels used in single request/response mechanics. Thanks to those properties they should be fairly lightweight and fast.

### Single consumer channel

Channel optimized for single consumer - especially usefull in actor scenarios.

### Multi consumer channel

Standard multi-write multi-read channel.

### Dual channel

Dual channel is combination of two channels combined in separate directions. This way we can have composable request-response channel.

```csharp
public class DualChannel<TReq, TRep> : IChannel<TReq>, IChannel<TRep> {
    public DualChannel(IChannel<TReq> forward, IChannel<TRep> backward) {}

    // Send message through forward channel and wait for the next response on the backward channel.
    // Forward input and backward output may not be correlated there. In this case is up to user to
    // guarantee correlation.
    public Task<TRep> Request(TReq request) {}

    // Combines two dual channels together, creating next dual channel in return.
    public DualChannel<TReq, TRep2> Link<TReq2, TRep2>(DualChannel<TReq2, TRep2> other) {}
}
```

### Priority / merge channel

Two types of possible channels, build internally of many different channels headed in the same direction. Difference is that PriorityChannel will always fetch messages in the same order (depending on sub-channel priority, while MergeChannel may be totally random).

```csharp
// other versions with more parameters arity
public class PriorityChannel<T1, T2> : IChannel<T1>, IChannel<T2>, IReceiveChannel<Either<T1, T2>> 
{
    public PriorityChannel(IChannel<T1> higherChannel, IChannel<T2> lowerChannel) {}

    // Either is tagged union - a struct that wraps one of the message types. This gives us an
    // uniform method to get any of the possible cases.
    public bool TryReceive(out Either<T1, T2> messageCase) {}
}
class PriorityChannel<T1, T2, T3> {}
class PriorityChannel<T1, T2, T3, T4> {}
```

This will be especially important in case of the actors, where priority channel for system signals is necessary.

Other interesting thing, we could do here, is that those channels could be partially linked.

```csharp
var chan1 = new PriorityChannel<int, int, string>();
// after linking channel, return new priority channel with unlinked parts
PriorityChannel<int, string> chan2 = chan1.LinkFirst(otherChannel);
```

### Broadcast channel

Broadcast channel can be linked on the read end to any number of messages, and will broadcast incoming message to all writeable channels.

### Network channel

Network channels are abstractions over underlying remote messaging protocol (i.e. TCP or UDP) in order to provide channel semantics between processes on two different machines.

```csharp
public class NetworkChannel<T>(ITransportProtocol<T> protocol) : IChannel<T>
{
}
```

> I think, that serialization should be encapsulated as part of the protocol. Most of the advantages could be reached from there i.e: recyclable byte buffers, context-aware serialization (because protocol knows what info is already known for both endpoints, it can optimize some of the serialized data in form of type maps/schemas).

## Exposing channels

Within a local process, there's nothing bad with using channels directly. However if we'd like to include them into serializable messages and provide some location transparency, we'd need to provide a way to reference them through a substitutes.

```csharp
interface IChannelRef<T> 
{
    void Send(T message);
}
```

This also is the reason, why all channels need to be addressable. To change channel into ref, we need a context in form of some server runtime.

```csharp
// server abstracts things like remoting and serialization
IChannelRef<int> ch = server.Register(new Channel<int>());
// serialize and deserialzie from stream
using (var stream = new MemoryStream()) 
{
    server.Serialize(ch, stream);
    ch = server.Deserialize(stream);
}
```

`IChannelRef<>` abstracts implementation details i.e. channel on local can be deserialized as ref for network channel to remote endpoint.

## Channel buffers

Channels will operate on their underlying buffers. Buffer describes, how message in channels are stored. Not all channels need to have them (see: `IVar`).

- `UnboundedBuffer()` has unlimited length.
- `FixedSizeBuffer(int size, OverflowStrategy strategy)` has fixed size and once reached, it will apply provided overflow strategy on the buffer.
- `ResizableBuffer(int initSize, int maxSize, OverflowStrategy strategy)` has some initial size, that can grow up to the certain maximum size, after which overflow strategy will be applied. Question: *do we need any buffer shrinking strategy here?*
