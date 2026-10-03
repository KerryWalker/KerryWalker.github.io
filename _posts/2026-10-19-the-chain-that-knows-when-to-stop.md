---
layout: post
title: "The Chain That Knows When to Stop: Chain of Responsibility"
pillar: patterns
series: design-patterns
excerpt: "I have written two things that both get called chains, and only one of them is Chain of Responsibility. The difference between them is whether anything in the chain is allowed to stop it."
tags:
  - design-patterns
  - csharp
  - dotnet
---

The app I work on has a search box in the top corner. One box. The user types something, presses enter, and what comes back might be a customer, an order, a site, an invoice, a contract or a product. Six kinds of record, six tables, six different formats of reference number.

There's no dropdown asking which one they meant. That was deliberate. The people using this have a reference written on a note in front of them and they want to type it in and get on with their day. Working out what sort of reference it is was supposed to be our problem, not theirs.

Which leaves the obvious question. How does one box decide?

## The version everybody writes first

A big `if`. Does it look like an order number? Search orders. Does it start with a letter and then six digits? Search customers. Otherwise try the site codes.

That holds up for about three record types. Then somebody adds a fourth whose format overlaps with the second, and now the order of your `if` branches is load bearing and nobody has written that down. Then a fifth needs hiding from most users until the feature it belongs to goes live, so there's a flag in the middle of the condition. By the sixth the method is a hundred lines, it lives in the page that calls it, and testing one branch means instantiating everything.

The thing all six have in common is small: look at the text, decide if it's yours, have a go. Everything else about them is different. That's a shape worth pulling out.

## One searcher per kind of record

Each one became a class on an abstract base.

```csharp
public abstract class Searcher : ISearcher
{
    protected ISearcher _successor;

    public void SetSuccessor(ISearcher searcher)
    {
        _successor = searcher;
    }

    protected abstract bool CanHandleSearch(string searchText);

    protected abstract Task<QuickSearchResult> ProcessSearch(
        string searchText, CancellationToken cancelToken);
}
```

Two abstract members. Does this look like mine, and if so, go and get it. A searcher knows one format and one table, and that's all it knows.

The successor is the interesting bit. Each searcher holds a reference to another searcher, and they get wired together when the page is built:

```csharp
productAndOrderSearch.SetSuccessor(customerSearch);
customerSearch.SetSuccessor(siteSearch);
siteSearch.SetSuccessor(orderSearch);
orderSearch.SetSuccessor(contractSearch);
contractSearch.SetSuccessor(invoiceSearch);
```

Six links in a line, declared in one place you can read top to bottom. The order is still load bearing, but now it's five lines of wiring rather than a hundred lines of nested conditions, and changing it is a reordering rather than surgery.

The page holds the first searcher and calls one method on it.

## The method that does the work

Everything lives on the base class. No searcher overrides it.

```csharp
public async Task<QuickSearchResult> HandleSearch(string searchText, CancellationToken cancelToken)
{
    QuickSearchResult searchResult = null;

    if ((!FeatureFlag.HasFeature || await _featureManager.IsEnabledAsync(FeatureFlag))
        && CanHandleSearch(searchText.Trim()))
    {
        searchResult = await ProcessSearch(searchText.Trim(), cancelToken);
    }
    if (searchResult?.Success ?? false)
    {
        return searchResult;
    }
    else if (_successor != null)
    {
        var innerResult = await _successor.HandleSearch(searchText, cancelToken);
        if (innerResult != null)
        {
            searchResult = innerResult;
        }
    }
    return searchResult;
}
```

That's the **Chain of Responsibility**. A request passed along a line of handlers, each of which may deal with it or pass it on, and the caller has no idea which one answered.

Three things in those twenty lines are worth pulling apart, because they're the difference between the version in the book and the version that survives contact with a real application.

## It stops

This line is the pattern:

```csharp
if (searchResult?.Success ?? false)
{
    return searchResult;
}
```

Found it, done, nobody else gets a go. The invoice searcher at the end of the line never runs if the customer searcher at position two came back with something. That isn't an optimisation, it's the defining property. A chain whose links are steps rather than candidates is not this pattern, and I'll come back to that.

## A link can be switched off

Look again at the condition:

```csharp
(!FeatureFlag.HasFeature || await _featureManager.IsEnabledAsync(FeatureFlag))
```

Most searchers don't have a flag, so `HasFeature` is false and the condition short circuits. The ones that do are skipped entirely when the flag is off, and the request goes straight past them to the next link.

The original description of this pattern has no concept of a handler being disabled this week. Real systems are full of them. Searchers get written before the feature they search for is switched on, and they sit in the chain doing nothing until somebody flips a toggle. Nothing else in the chain knows, and nothing else has to change. That's worth more than it sounds. Adding a link to a chain is already cheap, and being able to add a dormant one is cheaper still.

## "Can you handle it" is a guess, not a promise

This is the part I'd not seen in any explanation of the pattern before I wrote one.

Every tutorial has `CanHandle`. Handler says yes, handler does it, done. Which quietly assumes a handler can tell, up front and for certain, whether the request is its business.

Ours can't. `CanHandleSearch` is a cheap test on the shape of the string. It can only ever say "this could be one of mine". Whether it actually is one of ours needs a database round trip, and the answer might be no.

So look at what happens when a searcher says yes, goes and looks, and comes back empty. `searchResult.Success` is false. The `if` doesn't return. The chain carries on to the successor and the next one tries. A handler is allowed to accept a request, fail at it, and hand it on anyway.

That's why the terminating condition is `Success` and not `CanHandle`. The cheap guess decides whether it's worth looking. The expensive truth decides whether the chain is over. Collapsing those two into one boolean is what makes textbook examples read so neatly and real ones not work.

## Where you've already met this

Two places, and neither of them looks like a line of objects.

Exception handling is Chain of Responsibility built into the language. You throw, and the runtime walks up the call stack offering the exception to each frame in turn until one has a `catch` that matches. Every frame is a potential handler, most of them decline, the first one that accepts ends it, and the code that threw has no idea which one caught. That is the whole pattern, and nobody wired anything together to get it.

UI event bubbling is the other. A click lands on a button inside a panel inside a form. The button gets first refusal, and if it doesn't handle the event it carries on outward until something does. Same shape again, with the parent relationship standing in for the successor.

Both are worth noticing because they tell you when to reach for this deliberately. You want it when you have several candidates, you don't know which one applies until you ask, and the caller shouldn't have to care.

## In short

Chain of Responsibility is a line of handlers where any one of them is allowed to end the line.

Keep the "do I handle this" test cheap, because it is only ever a guess, and let the real result decide whether the chain stops. Collapsing those two into a single boolean is what makes textbook examples read so neatly and real ones fall over. Feature flags on individual links cost almost nothing, and being able to add a link that lies dormant until someone flips a toggle is a good part of why the pattern keeps earning its place.

One thing I deliberately left hanging. Everything above turns on a link being allowed to stop the chain, and I said I would come back to it.

In another codebase I wrote something that looks almost identical. A `SetNext` method, a field holding the next one, objects wired in a line at startup. You could put the two files side by side and struggle to tell them apart. But none of its links is an alternative to any other. They all take a turn, in order, each one working on whatever the last one produced, and it is not this pattern at all. That one is next.

