# Testing actors

Given actor-as-a-function trait, testing actors looks faily simple. When take a look at that signature, you may notice that we don't need to mock the whole actor runtime, only a necessary parts of the actor context itself:

```csharp
Receive<Greet> greeter = async (ctx, either) => {
    if (either.Case == EitherCase.Right) 
    {
        var greet = either.Right;
        greet.ReplyTo.Tell("hello " + greet.Whom);
    }
    return Effects.Stay;
}

// run test 
using(var ch = new VarChannel<string>())
{
    await greeter(null, Either.Right(new Greet("John", ch)));
    var reply = await ch;
    Assert.Equals("hello John", reply);
}
```

Since first parameter is `IActorContext`, it can be mocked if necessary. With this, making fast responsive tests become trivial.

## Testing clusters