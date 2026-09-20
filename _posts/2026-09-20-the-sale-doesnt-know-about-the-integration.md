---
layout: post
published: false
title: The Sale Doesn't Know About the Integration
pillar: patterns
series: design-patterns
excerpt: I needed new things to happen when old things happened, without editing the old things. Domain events and the Observer pattern, in code that had been running for twenty years before I turned up with an idea.
tags:
  - design-patterns
  - csharp
  - dotnet
---

A while back I picked up a piece of integration work. Another system needed telling when certain things happened in ours — an order completes, a stock item changes — so it could keep its own records in step.

The things that needed to trigger it were all over the place, and most of them sat in code that had been working quietly for years. Code I didn't fancy opening up and editing just to bolt a new feature onto the side of it.

That's the problem this post is about. Not "how do I call the other API", which is the easy half. How do I make new things happen when existing things happen, without going round rewriting all the existing things?

## The Obvious Way, and Why I Didn't

The direct approach is to find the place an order gets completed and add the call there:

```csharp
CompleteTheOrder();
_partnerIntegration.NotifyOrderCompleted(orderId);
```

One line. Nothing wrong with it on day one.

But the method that completes an order is long-standing code that lots of things rely on, and it's not the only place this needs to happen. Do that half a dozen times and you've threaded a brand-new integration through half a dozen bits of old code that previously had nothing to do with each other. Every one of them now needs to know the integration exists. Every one of them breaks if it changes.

And then somebody asks for a second thing to happen on order completion — an email, say — and you go back and edit all six again.

## Raising a Hand Instead

So the code doesn't call the integration. It announces that something happened, and has no idea who's listening.

An event is just a small class holding what happened:

```csharp
public class OrderCompletedArgs : IDomainEventArgs
{
    public OrderCompletedArgs(int orderId, int siteId) { ... }

    public int OrderId { get; }
    public int SiteId { get; }
}
```

The code that completes the order raises it:

```csharp
var args = new OrderCompletedArgs(orderId, siteId);
await _dispatcher.DispatchAsync(new[] { args });
```

And that's the entire change to the existing code. It doesn't reference the integration, doesn't know what a partner API is, and won't need touching again when the next requirement lands.

Anything that cares implements a handler:

```csharp
internal sealed class OrderCompletedHandler : IDomainEventHandler<OrderCompletedArgs>
{
    public async Task<ResultString> Handle(
        OrderCompletedArgs domainEvent, CancellationToken cancellationToken = default)
    {
        // call the partner API
    }
}
```

And says so once, at startup:

```csharp
services.RegisterDomainEventHandler<OrderCompletedArgs, OrderCompletedHandler>();
```

The dispatcher takes an event, finds every handler registered for that type, and runs them. That's the whole mechanism.

This is the **Observer** pattern, or publish-subscribe if you prefer that name. One thing announces, any number of things listen, and the announcer never learns who they were. The important word is *never*: the order-completion code has no way to find out what handled its event, which is precisely why adding the email later won't mean going back and editing it.

## Where You've Already Met This

You have been doing this for years, in the smallest possible way, every time you write:

```csharp
button.Click += SaveButton_Clicked;
```

The button doesn't know what happens when it's clicked. It has a list of things to call and calls them, and you can add another without modifying the button. Same idea, just scaled up: instead of a control with one event, you have an application-wide dispatcher and handlers registered in your DI container.

Every message queue you've used is the same pattern again, with the network in the middle.

## The Two Awkward Questions

Announcing something is easy. The interesting decisions are *when* you announce it and *what happens when a listener falls over*.

**When.** Ours dispatch after the database transaction has committed, not inside it:

```csharp
    transScope.Complete();
}
// transaction is committed, now tell everyone
await _dispatcher.DispatchAsync(new[] { args });
```

It matters more than it looks. Raise the event inside the transaction and your handler runs while the data is still uncommitted — so it can tell the outside world about an order that is about to be rolled back, and you've just told another company about something that never happened. Announce things once they're true.

**Failure.** The other decision is in a comment in our code:

```csharp
// Don't fail the order completion if the sync fails, just log it
```

If the partner API is having a bad afternoon, the order still completes. The alternative — failing the customer's order because a third party is down — is obviously worse. So the handler logs, and something else picks up the pieces later.

That won't always be the right call. Sometimes a failed handler *should* stop everything. But it's a decision you have to make on purpose, because the default — an exception bubbling up out of a handler nobody knew was running — is the worst of both worlds.

## The Seam With the Old Code

One honest wrinkle, because this is a real codebase rather than a sample.

The dispatcher is a modern service, resolved from the DI container. The order-completion code is not — it predates the container by a long way, and it's synchronous. So at that particular join, the old code reaches out and fetches the dispatcher rather than being handed it, and waits on the async call rather than awaiting it.

Neither is how I'd write it fresh. Both are what letting the old code stay exactly as it is actually costs. The alternative was to modernise a twenty-year-old order pipeline before I could send a single message, which wasn't the job I'd been given. Every pattern is easier in a new codebase, and almost nobody gets to use one.

## What It Bought

The integration went in without the existing code learning a thing about it. Since then, new handlers have been added to events that were already being raised, and the code raising them hasn't been reopened once.

That's the pattern's whole promise, and it's a modest one. It doesn't make anything faster or cleverer. It just means the next person who needs something to happen when an order completes can add it without reading, understanding and carefully not breaking the code that completes orders.

There's more to it — ordering, retries, what to do when a handler needs to run outside the request, and a base class that stops handlers being written wrong, which is a post of its own. But this should cover the basics.
