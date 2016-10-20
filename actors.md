# Actors

Given already what was written in [Channels](./channels.md) and [Processes](./processes.md) sections, we can now define actors as a composition of those two.

Actors are described in context of their receiver logic:

```csharp
public delegate ValueTask<IEffect> Receive<T>(IActor context, Either<T, ISignal> message);
```

Effect describes how the message execution resulted in change in actor's behavior. Few examples:

- `Unhandled` - message was not handled.
- `Become<T>(Receive<T> next)` changes current behavior to the `next` one.
- `Stop` stops an actor.

## Protocols

Effect describes change of processing fragment. In ideal world this would allow us to compose type-safe protocols, not only interfaces. Example: 

```csharp
interface IProtocol<T> 
{
    void Init();
    void Process(T message);
    void Close()
}
```

Problem with this interface is that we cannot guarantee that Process won't be called before Init or after Close (keep in mind, that this is fairly simple example). Thinking in context of process and effects we could describe that protocol as:

```csharp
// Become<T,Close> means that it can accept either T => Become<T> or Close to escape current step 
Protocol<Init<Become<T,Close>> protocol;
Protocol<Become<T,Close>> initialized = protocol.Accept(new Init());
Protocol<Become<T,Close>> processed1Message = initialized.Accept(new T());
Protocol<Become<T,Close>> processed2Messages = initialized.Accept(new T());
Protocol<Unit> closed = initialized.Accept(new Close());
```

Pseudocode above works in the similar way to Haskell HLists. If .NET type system would support that, we could provide compile time guarantee that protocol transitions will be executed in correct order. 

## Composing actor behaviors

Instead of using inheritance we could use composition - similar to how the Services and Filters works in Finagle. Example:

```csharp

public delegate ValueTask<Effect> Filter<TIn, TOut>(IActor context, Either<TIn, ISignal> msg, Receive<TOut> receive);

// compose two Filters
public static Filter<TIn, TOut2> Then<TIn, TOut, TOut2>(this Filter<TIn, TOut> filter, Filter<TOut, TOut2> next)
    => (ctx, msg, receive) => filter(ctx, msg, (ctx2, msg2) => next(ctx2, msg2));

// compose filter with receive
public static Receive<TIn> Bind<TIn, TOut>(this Filter<TIn, TOut> filter, Receive<TOut> receive)
    => (ctx, msg) => filter(ctx, msg, receive);
```

Given those definitions, we could have something more real life:

```csharp
interface IMessage {}
class Increment : IMessage, IRepyable<int> {
    public IChannelRef<int> ReplyTo { get; }
    public Increment(string who, IChannelRef<int> replyTo) { Who = who; ReplyTo = replyTo; }
}
class Incremented : IMessage {}
var eventsourced = new Eventsourced<IMessage>();

Receive<IMessage> myActor(int state) =>
    async (ctx, either) => {
        switch(either.Case) {
            // check if message is signal
            case EitherCase.Fist: return Effects.Stay;
            // check if message is command or event
            case EitherCase.Second:
                IMessage msg = either.Second;
                if (msg is Increment) {
                    var command = msg as Increment;
                    // persist a new event 
                    await eventsourced.Persist(new Incremented());
                    // update a state and respond back
                    var newState = state + 1;
                    command.ReplyTo(newState);
                    return Effects.Become(myActor(newState));

                } else
                // update a state
                return Effects.Become(myActor(state+1));
        }
    };

// combine filter with behavior
var receive = eventsourced.Filter.Bind(myActor(0));
```

Proposed F# API:

```fsharp
type Message =
    | Increment of IChannelRef<int>
    | Incremented

let eventsourced = Eventsourced()
let myActor state msg = actor {
    match msg with
    | Sig(_, _) -> return! myActor state
    | Msg(ctx, message) ->
        match message with
        | Increment replyTo ->
            do! eventsourced.Persist Incremented
            replyTo <! (state+1)
            return! myActor (state+1)
        | Incremented ->
            return! myActor (state+1)
}
let receive = eventsourced <|> myActor 0 
```

## [Spawning actors](./spawning-actors.md)


## Communication mechanisms