---
layout: post
title: "Mission Asyncable - The Basics of Asyncs - Part 1: Cancellation"
date: 2020-12-20-00:00:00 -0000
comments: true
published: true
excerpt_separator: <!--more-->
---

_This post is part of F# Advent Calendar 2020. Check out other posts [here](https://sergeytihon.com/2020/10/22/f-advent-calendar-in-english-2020/)._

## Motivation
On my team, we've been running into a few microservices stalling and other production issues, due to some nuanced assumptions around Async and cancellations. Knowing the fundamentals is important, and I thought it'd be a useful experience to review them, perhaps from a different angle.

## CancellationTokens
One of the more "invisible" aspects of Async workflows in F# are [CancellationTokens](https://docs.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken?view=net-5.0). They are constructs that are implicitly passed through Async workflows to enable cooperative cancellation.

Let's break that down. They're _implicitly passed through_ in the sense that a parent Async's `CancellationToken` is forwarded to child workflows it spawns. By spawns, I mean a child workflow that is handed over control to, usually via `let!` or `do!` on an `Async<'T>` (Note: this propagation can be circumvented, as will be explained below). By _cooperative_, we mean that a "well-behaved" Async workflow is supposed to inspect the token to see if cancellation's been requested. It behaves like a good citizen by cancelling the current workflow as quickly as possible, but this is only an expectation, there are no guarantees it will do so.

<!--more-->

## Secret Async Man
Think of your Async workflows as a set of special agents from Mission Impossible; you have your fearless leader Ethan Hunt (played by Tom Cruise), your main infiltrators, backups, surveillance, etc. They all have their smartphones/watches synced up to some channel; it could be a top secret radio channel, a conference call, or if they’re cost-cutting even just a shared WhatsApp Group. If the mission goes awry, then Ethan will shout/type "ABORT!" on the channel, expecting everyone to drop whatever they're doing and leave their posts. (Note: this post is best read while listening to the [Mission Impossible theme song from the first movie](https://www.youtube.com/watch?v=XAYhNHhxN0A).)

![Abort](/assets/abort.png)

Now, they may not get the message or they may go rogue, and end up not aborting (that usually doesn't go well in the movies…). Or depending on the nature of the mission, maybe anyone can shout "ABORT!". Or perhaps there are sub-groups, like infiltrators, that need to step back, but surveillance can continue. All of these models are expressible as Async workflows using CancellationTokens.

There are 2 main concepts when talking about `Async` cancellation: [CancellationTokenSource](https://docs.microsoft.com/en-us/dotnet/api/system.threading.cancellationtokensource?view=net-5.0) and [CancellationToken](https://docs.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken?view=net-5.0). `CancellationTokenSource` is what backs a `CancellationToken`; it signals to the token that it should be cancelled. A `CancellationToken` is a struct that is passed around for `Async` workflows to detect that cancellation was requested. This [stackoverflow post](https://stackoverflow.com/questions/14215784/why-cancellationtoken-is-separate-from-cancellationtokensource) has a good explanation of why there are 2 classes instead of just one, and as it says, the short answer is separation of concerns.

Using our above mental model, you can think of them as:
- `CancellationTokenSource` = ability to shout Abort! (i.e. ability to request cancellation)
- `CancellationToken` = ability to hear someone shouting Abort! (i.e. ability to check if cancellation was requested)

With that mental model, let's try to express each of the above ideas as Async workflows.

## Examples
Note: In all examples for this post, I'm _deliberately_ not using `Async.Parallel`; it has more nuanced cancellation behavior that we'll explore in part 2. I'll be starting  Asyncs as Tasks instead. `Task`, which is part of the [.NET TPL (Task Parallel Library)](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/task-parallel-library-tpl), is a similar abstraction to `Async` (with some important differences we won't get into here).

### Shared code

This is shared code that will be used in the examples below.

```fsharp
let customCts = new CancellationTokenSource()

// A bit redundant, trying to keep code simple
let cancelDefaultTokenAsync sleepMs = async {
    do! Async.Sleep sleepMs
    Async.CancelDefaultToken()    
}

// cancels tokens related to customCts
let cancelCustomTokenAsync sleepMs = async {
    do! Async.Sleep sleepMs
    customCts.Cancel()
}

let whenAll (tasks: IEnumerable<Task>) =
    Task.WhenAll(tasks).ContinueWith(Action<Task>(fun t -> printfn "Done"))

let infiltrators teamNumber = async {
    printfn "Infiltrator team %d: Starting" teamNumber
    for i in 1..5 do        
        printfn "Team %d Infiltrating" teamNumber
        do! Async.Sleep 1000
    
    printfn "Team %d infiltration complete" teamNumber
}

let surveillance teamNumber = async {
    printfn "Surveillance team %d: Starting" teamNumber
    do! infiltrators teamNumber
    for i in 1..5 do        
        printfn "Team %d Surveilling" teamNumber
        do! Async.Sleep 1000
        
    printfn "Team %d surveillance complete" teamNumber
}

let missionLeader teamNumber = async {
    printfn "Mission Leader %d: Starting Mission" teamNumber
    do! surveillance teamNumber
    for i in 1..5 do        
        printfn "Leader %d Observing mission" teamNumber
        do! Async.Sleep 1000
        
    printfn "Leader %d mission complete" teamNumber
}
```

### Implicit CancellationToken passing

Code
```fsharp
let t1 = missionLeader 1 |> Async.StartAsTask
let cancelDefault = cancelDefaultTokenAsync 3000 |> Async.StartAsTask
whenAll([t1; cancelDefault])
```

Output
```
Mission Leader 1: Starting Mission
Surveillance team 1: Starting
Infiltrator team 1: Starting
Team 1 Infiltrating
Team 1 Infiltrating
Team 1 Infiltrating
Done
```

We're running an `Async` task with nested workflows (surveillance/infiltrators), and an `Async` that will cancel the default token (we'll get to that soon) after 3 seconds. You can see "infiltrating" printed thrice, after which all workflows are cancelled. `whenAll` will print "Done" when all Tasks are done.

This is what is meant by _implicitly passed through_. Each child `Async` is working off of the same `CancellationToken`, so once cancellation is requested (i.e. ABORT!), all the Asyncs stop working and exit.

Notice that the cancel method is calling `Async.CancelDefaultToken`. We didn't explicitly pass a `CancellationToken`, but we did observe cancellation. As you may have surmised, there's a default `CancellationToken` that's passed into your Async when it's started. This token is unique in that it's shared by default by all `Async` workflows, but other than that there's nothing special about it. We could've passed in our own, as long as we _remember to trigger cancellation via that same_ `CancellationTokenSource`. (Keep in mind the name CancelDefault**Token** is slightly misleading; we're technically requesting cancellation on the `CancellationTokenSource` that backs that `CancellationToken`.)

### Custom CancellationToken 

Code
```fsharp
let t1 = Async.StartAsTask(missionLeader 1, cancellationToken=customCts.Token)
let cancelCustom = cancelCustomTokenAsync 3000 |> Async.StartAsTask
whenAll([t1; cancelCustom])
```

Output
```
Mission Leader 1: Starting Mission
Surveillance team 1: Starting
Infiltrator team 1: Starting
Team 1 Infiltrating
Team 1 Infiltrating
Team 1 Infiltrating
Done
```

This has the same behavior and output as above, just notice we're passing in a custom token and cancelling on that same token.

### DefaultCancellationToken cancelling many workflows

`Async.DefaultCancellationToken` is a static property of Async, which means that every Async workflow you invoke that doesn't specify a custom `CancellationToken` **will all share this default one**. This means a cancellation in one workflow can cause other workflows to cancel as well.

Code

```fsharp
let t1 = infiltrators 1 |> Async.StartAsTask
let t2 = infiltrators 2 |> Async.StartAsTask
let cancelDefault = cancelDefaultTokenAsync 3000 |> Async.StartAsTask
whenAll([t1; t2; cancelDefault])
```

Output

```
Infiltrator team 1: Starting
Team 1 Infiltrating
Infiltrator team 2: Starting
Team 2 Infiltrating
Team 2 Infiltrating
Team 1 Infiltrating
Team 2 Infiltrating
Team 1 Infiltrating
Done
```

Notice how cancellation on the `DefaultCancellationToken` took out both teams (and that's not a good attack plan is it?). This could result in unexpected behavior if you're not aware of it.

### Custom CancellationToken while cancelling default

You can invoke a workflow with a custom `CancellationToken`, but if you're still cancelling on the default one, that workflow won't cancel.

Code

```fsharp
let t1 = infiltrators 1 |> Async.StartAsTask
let t2 = Async.StartAsTask(infiltrators 2, cancellationToken=customCts.Token)
let cancelDefault = cancelDefaultTokenAsync 3000 |> Async.StartAsTask
whenAll([t1; t2; cancelDefault])
```

Output
```
Infiltrator team 1: Starting
Infiltrator team 2: Starting
Team 2 Infiltrating
Team 1 Infiltrating
Team 2 Infiltrating
Team 1 Infiltrating
Team 2 Infiltrating
Team 1 Infiltrating
Team 2 Infiltrating
Team 2 Infiltrating
Team 2 infiltration complete
Done
```

Team 1 cancels, but team 2 doesn't.

### Tasks and Tokens

If your `Async` workflows are inter-operating with TPL `Task`, in order to support cancellation (hint: you should), then you need to explicitly pass CancellationTokens when the `Task` is started. Typically, `Async`/`Task` functions that do I/O or sleep will have overloads that accept `CancelltionToken`.

Code

```fsharp
let t1 = infiltrators 1 |> Async.StartAsTask
let t2 = Task.Delay(5000)
             .ContinueWith(Func<Task, unit>(fun t -> printfn "Team 2 as task done"))
             // needed to return Task<unit>
let cancelDefault = cancelDefaultTokenAsync 3000 |> Async.StartAsTask
whenAll([t1; t2; cancelDefault])
```

Output

```
Infiltrator team 1: Starting
Team 1 Infiltrating
Team 1 Infiltrating
Team 1 Infiltrating
<2 seconds pass>
Team 2 as task done
Done
```

Notice that infiltrators cancel after 3 seconds due to `cancelDefaultTokenAsync`, but "Team 2 as task done" still finishes after 5 seconds, because it hasn't been passed a token.

We can get have the TPL `Task` honor the cancellation request by using the overload where we can supply it:

Code

```fsharp
let t1 = infiltrators 1 |> Async.StartAsTask
let t2 = Task.Delay(5000, Async.DefaultCancellationToken).ContinueWith(Func<Task, unit>(fun t -> printfn "Team 2 as task done"))
let cancelDefault = cancelDefaultTokenAsync 3000 |> Async.StartAsTask
whenAll([t1; t2; cancelDefault])
```

Output

```
Infiltrator team 1: Starting
Team 1 Infiltrating
Team 1 Infiltrating
Team 1 Infiltrating
Team 2 as task done (<-- happens immediately after)
Done
```

The same output is printed, but when you run it you'll notice that "Team 2 as task done" prints as soon as the `DefaultCancellationToken` is cancelled, without the 2 second delay.

This can lead to surprises when you have an `Async` workflow that awaits a `Task` (using `Async.AwaitTask`), if the `CancellationToken` isn't passed in at `Task` creation time. If you're relying on expected cancellation behavior, it's important to know what's backing some random `Async<'T>` that's being awaited in the workflow. A Task that didn't have the `CancellationToken` passed in can lead to unexpected behavior, especially if that `Task` is long/infinite running (this relates back to the stalling services in production).

### Cooperation
How does an individual `Async` workflow know that it's time to cancel? Well, remember what we said above: CancellationTokens allow for a _cooperative_ cancellation model. That means once an Async's `CancellationToken` is signaled, it's expected to cooperate and stop its execution, but this isn't guaranteed. We're going to call these non-cooperative workflows the rogue agents of the Async world.

Usually, I/O operations that are returned as `Async`/`Task` will respect cancellation, as will `Async.Sleep`/`Task.Delay`. However, if an `Async`/`Task` is doing a compute-heavy operation, it should be checking for cancellation periodically; this is done by inspecting the `IsCancellationRequested` on the `CancellationToken`.

Code

```fsharp
let rogueInfiltrators teamNumber =     
    let rec busyWait waitMs (token: CancellationToken) =
        if waitMs > 0L then
            if not token.IsCancellationRequested then // comment this line out to see the cancellation request ignored
                printfn "Team %d Infiltrating" teamNumber
                
                // this is a terrible way to busy wait
                let sw = Stopwatch.StartNew()
                while sw.ElapsedMilliseconds < 1000L do
                    let mutable x = 1
                    for i in 1..10000 do
                        x <- x + 1
                        
                busyWait (waitMs-sw.ElapsedMilliseconds) token
      
    async {
        printfn "Rogue Infiltrator team %d: Starting" teamNumber
        let! token = Async.CancellationToken
        busyWait 5000L token    
        printfn "Team %d infiltration complete" teamNumber
    }

let t1 = rogueInfiltrators 1 |> Async.StartAsTask
let cancelDefault = cancelDefaultTokenAsync 1000 |> Async.StartAsTask
whenAll([t1; cancelDefault])
```

Output

```
Rogue Infiltrator team 1: Starting
Team 1 Infiltrating
Team 1 infiltration complete
Done
```

This code loops with a meaningless computation to simulate some CPU intensive work. It'll only continue if the `IsCancellationRequested` property is false, and the wait time is positive. If you comment out the line to check cancellation requests, you'll see it won't respect cancellation at all.

But wait, I hear you say, you've snuck in some new code. What is this `Async.CancellationToken`? This returns what is referred to as the _ambient_ `CancellationToken` assigned to that Async, i.e. the `CancellationToken` fed through the tree of computations in a nested Async workflow. It's what's being passed from parent to child Asyncs, and what we should be using to check cancellation on, since we can't assume it'll always be the default token.

### Sub groups

As we mentioned above, what if we only want to cancel a certain group of Asyncs, but leave the rest running? That's where the `CancellationTokenSource.CreateLinkedTokenSource` method comes in. It creates a new `CancellationTokenSource` that will cancel if the passed-in token cancels. Since it's a separate source however, if it's cancelled, then only workflows working off of that source will be cancelled. That's a mouthful, so let's look at some examples. (Notice we have some new code for `linkedCts` and a function for cancelling tokens related to it).

Code

```fsharp
let linkedCts = CancellationTokenSource.CreateLinkedTokenSource(Async.DefaultCancellationToken)

let t1 = Async.StartAsTask(infiltrators 1, cancellationToken=linkedCts.Token)
let cancelDefault = cancelDefaultTokenAsync 3000 |> Async.StartAsTask
whenAll([t1; cancelDefault])
```

Output

```
Infiltrator team 1: Starting
Team 1 Infiltrating
Team 1 Infiltrating
Team 1 Infiltrating
Done
```

`linkedCts` is linked to the `DefaultCancellationToken`, so when the default token cancels, it and all Asyncs running off of it cancel as well.

Code

```fsharp
// cancels tokens related to linkedCts
let cancelLinkedTokenAsync sleepMs = async {
    do! Async.Sleep sleepMs
    linkedCts.Cancel()
}

let t1 = Async.StartAsTask(infiltrators 1, cancellationToken=linkedCts.Token)
let t2 = infiltrators 2 |> Async.StartAsTask
let cancelLinked = cancelLinkedTokenAsync 3000 |> Async.StartAsTask // notice we're cancelling only the linked one here!
whenAll([t1; t2; cancelLinked])
```

Output

```
Infiltrator team 1: Starting
Infiltrator team 2: Starting
Team 1 Infiltrating
Team 2 Infiltrating
Team 1 Infiltrating
Team 2 Infiltrating
Team 2 Infiltrating
Team 1 Infiltrating
Team 2 Infiltrating
Team 2 Infiltrating
Team 2 infiltration complete
Done
```

But if **only** `linkedCts` is cancelled, then only team 1 (which is running off of it) cancels; team 2 isn't aware of any cancellation.

### Where can I specify CancellationTokens?
Every Async workflow has an _ambient_ `CancellationToken` associated with it; if you don't supply a value for `CancellationToken` then `Async.DefaultCancellationToken` is used (except for `StartChildAsTask`). The functions which start them are documented [here](https://fsharp.github.io/fsharp-core-docs/reference/fsharp-control-fsharpasync.html#section0), and copied below.
	
* `Async.RunSynchronously(computation, ?timeout, ?cancellationToken)`
* `Async.Start(computation, ?cancellationToken)`
* `Async.StartAsTask(computation, ?taskCreationOptions, ?cancellationToken)`
* `Async.StartImmediate(computation, ?cancellationToken)`
* `Async.StartImmediateAsTask(computation, ?cancellationToken)`
* `Async.StartWithContinuations(computation, continuation, exceptionContinuation, cancellationContinuation, ?cancellationToken)`

_If you use any of these functions to start an Async workflow inside another Async workflow, they won't necessarily share the same CancellationToken (unless they're all using the default one)_.

Notice that one of the functions, `Async.StartChildAsTask(computation, ?taskCreationOptions)`, doesn't take in a `CancellationToken`. This is because it's a child of the current `Async` workflow (like we saw above) just wrapped in a Task, which mean it'll inherit the parent Async's `CancellationToken`.

# Conclusion
We've looked at the cancellation aspect of `Async` workflows:
* how default tokens are passed around
* how to pass in custom cancellation tokens
* how `Async` cancellation interacts with TPL `Task`
* how to make sure a compute-heavy workflow respects cancellation requests
* how to use a `LinkedTokenSource` to create sub-groups of independently cancellable workflows
* how they suspiciously relate to the world of espionage

That's it for this time. In part 2 we'll go over some aspects of `Async.Parallel` and how it relates to what we've learned here. This post will self destruct in 5 seconds.