---
layout: post
title: "The Base Class That Won't Let You Forget: Template Method"
pillar: patterns
series: design-patterns
excerpt: Two checks that every event handler had to do before it did anything, written out by hand in each one, until somebody wrote a handler and missed them. The Template Method pattern, and why it isn't the same as putting shared code in a base class.
tags:
  - design-patterns
  - csharp
  - dotnet
---

[Last time]({% post_url 2026-09-20-the-sale-doesnt-know-about-the-integration %}) I wrote about the event system we use to tell another company's API when things happen in ours. Something completes an order, an event goes out, and any number of handlers pick it up without the order code knowing they exist.

What I skipped over was that a handler can't just run. It has to check a couple of things first, and that turned into a small lesson about where shared code belongs.

## The Two Checks

Before a handler does anything, two things have to be true.

The first is a feature flag. Most of this went out behind one so we could turn it off in a hurry if the partner's API started misbehaving, without shipping anything.

The second is whether the customer is actually entitled to it. Not everyone has the integration. Usually it comes with having it switched on for their account, sometimes it's an extra on top. Either way, sending a customer's data to a third party they haven't signed up to is not something you want happening by accident.

So every handler starts the same way: is the flag on, does this customer have it, and if either answer is no, stop quietly and report success. Nothing went wrong, there was just nothing to do.

## How It Went Wrong

The first handlers had those checks written into them. Two of them, then three, each opening with the same few lines before getting to the interesting part. It looked fine. It was fine, right up until it wasn't.

A colleague added some new handlers. Perfectly good code, doing exactly what it was supposed to, except it went straight in and did the work. The checks weren't there. Nothing in the codebase said they had to be, and nothing complained when they weren't, because a handler is just a class with a method on it. Write the method, register it, done.

It got picked up in the pull request review, so nothing ever reached a customer. But that's the wrong thing to take comfort from. Catching it meant somebody reading the diff happened to know those lines were supposed to be there and happened to notice they weren't, on a change that otherwise looked completely reasonable. That worked once. It's not a thing you'd want to rely on every time.

Because it wasn't a careless mistake. The way you wrote a handler was "look at an existing one and do what it does", and that only holds while everyone notices every line that matters. The checks were a convention, and conventions are what people miss when they're concentrating on something else.

## Moving the Checks Somewhere They Can't Be Missed

The fix was to stop asking handlers to remember. The base class does the checking, and it decides when the handler's own code runs:

```csharp
public abstract class HandlerBase<T> : IDomainEventHandler<T> where T : IDomainEventArgs
{
    private readonly IFeatureManager _featureManager;
    private readonly ISubscriptionProvider _subscriptionProvider;

    protected HandlerBase(IFeatureManager featureManager, ISubscriptionProvider subscriptionProvider)
    {
        _featureManager = featureManager;
        _subscriptionProvider = subscriptionProvider;
    }

    public async Task<Result> Handle(T domainEvent, CancellationToken cancellationToken = default)
    {
        if (!await CanHandle(domainEvent))
        {
            return Result.DefaultSuccess;
        }

        return await Execute(domainEvent, cancellationToken);
    }

    protected abstract Task<Result> Execute(T domainEvent, CancellationToken cancellationToken = default);

    public async Task<bool> CanHandle(T domainEvent)
    {
        if (FeatureFlag.HasFeature && !await _featureManager.IsEnabledAsync(FeatureFlag.Name))
        {
            return false;
        }

        if (!await _subscriptionProvider.HasSubscription())
        {
            return false;
        }

        return true;
    }

    public virtual AppFeatureFlag FeatureFlag => AppFeatureFlag.None;
}
```

`Handle` is the method the dispatcher calls, and it's the only one that decides the order: check first, then run. `Execute` is abstract, so a handler has to supply it, and that's the only thing a handler supplies.

A handler now looks like this, and there is nowhere left to forget anything:

```csharp
internal sealed class OrderCompletedHandler : HandlerBase<OrderCompletedArgs>
{
    public OrderCompletedHandler(
        IFeatureManager featureManager, ISubscriptionProvider subscriptionProvider)
        : base(featureManager, subscriptionProvider) { }

    protected override async Task<Result> Execute(
        OrderCompletedArgs domainEvent, CancellationToken cancellationToken = default)
    {
        // just the work
    }

    public override AppFeatureFlag FeatureFlag =>
        AppFeatureFlag.For(ConfigurationEntries.SendCompletedOrdersToPartner);
}
```

That's the **Template Method** pattern. A base class writes down the shape of the algorithm and leaves holes in it. Subclasses fill the holes. They don't get to choose when their bit runs, or what happens before it.

## It Isn't Just Shared Code in a Base Class

This is the bit I think gets missed, and I'd have missed it myself a few years ago.

In [the factory post]({% post_url 2026-09-16-i-called-it-a-factory %}) I mentioned a set of base classes under the pivot helpers that grew out of noticing the same code in two places. Those base classes hold a nasty thirty-line method for turning selected grid cells back into record ids, written once and inherited by all eighteen. That's genuinely useful. It's also not Template Method.

The difference is who's in charge. With ordinary inheritance, the base class offers something and the subclass decides whether to call it. A pivot helper that never calls the drill-down method still compiles, still runs, and nobody finds out until a user selects some cells and nothing happens.

With Template Method, the base class calls *you*. `Execute` can't run early, can't run instead of the checks, and can't run at all if `CanHandle` says no, because a handler never gets to make that decision. The order lives in one place and there's no way to write a handler that disagrees with it.

Sharing code saves you typing. Template Method saves you from yourself. They look almost identical on a class diagram and they solve completely different problems.

## The Optional Hole

One detail in that base class is worth pointing out, because it's the other half of the pattern.

`Execute` is `abstract`. You must supply it, and the compiler says so.

`FeatureFlag` is `virtual`, defaulting to `AppFeatureFlag.None`. You may supply it, and if you don't, the handler just runs whenever the subscription check passes.

That's the choice you make for every step in the skeleton: compulsory or optional. Get it wrong in the compulsory direction and you force every subclass to write a method it doesn't care about. Get it wrong in the optional direction and you're back where you started, relying on people to remember. We made the subscription check compulsory and unskippable because that one has a customer on the end of it, and the flag optional because plenty of handlers don't need one.

## Where You've Already Met This

All over the place, usually without the name attached:

- A `BackgroundService` in .NET. You override `ExecuteAsync`. You don't decide when it's called, or what happens around it on shutdown.
- `OnModelCreating` on an EF Core `DbContext`. EF owns the model-building sequence and calls you at the point where your bit belongs.
- Every `On`-something method you've ever overridden in a UI framework. `OnPaint` doesn't run because you called it.

The common thread is that the framework owns the order and you own one step. Every time you've written `override`, some base class somewhere was doing to you what `HandlerBase` does to our handlers.

## What It Costs

It's inheritance, so it comes with inheritance's problems. You get one base class and that's your lot, so if a handler ever needs to be something else as well, you're stuck. Adding a step to the skeleton changes it for everybody at once, which is the entire point right up until the day it isn't.

And it only works if the base class genuinely knows the right order. If half your handlers want the checks and half don't, you haven't found a template, you've found two different things that happen to look similar.

For this, though, the sums were easy. Every handler wants both checks, in that order, every time. The only question was whether remembering that was going to be each developer's job or the code's, and we'd already run the experiment on that one.

## In Short

If the same lines have to appear at the start of every implementation, and something bad happens when they don't, don't share the lines. Take the decision away. Put the order in a base class, leave a hole for the part that varies, and make the hole the only thing anyone can fill in.

This is the plain-English version. There's more to it, and the moment you have more than a couple of optional steps it's worth asking whether you want a pipeline of small pieces instead of one base class. But this should cover the basics.
