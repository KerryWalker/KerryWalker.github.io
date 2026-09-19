---
layout: post
title: I Called It a Factory Without Knowing If It Was One
excerpt: One screen, a drop-down, and eighteen different ways of analysing the data behind it. The Factory pattern in real code — and why the thing everyone calls a factory isn't in the famous book at all.
tags:
  - design-patterns
  - csharp
  - dotnet
---

Years ago I wrote a class, called it a factory, and never felt entirely sure I'd earned the name. I'd read the definitions, and the definitions didn't quite describe what I'd built. It worked, everyone knew what it did, so I left it alone and got on with something else.

Writing this series made me go back and check. It turns out I was right to be unsure, and right to call it a factory anyway. Here's the code, and here's what the books actually say.

## The Problem

The request was for a set of pivot grids. If you've used a pivot table in Excel you know the idea: data in the middle, drag a field into the rows, drag another into the columns, and it regroups and totals everything for you. Ours sat in a desktop app on top of a grid control, and the whole point was that the user could move the ordering and grouping about and collate by almost any field.

There wasn't one pivot, there were eighteen. Different datasets, and different views of the same dataset. What I got was one screen with a drop-down: pick an analysis and everything else loads with it — the data, the columns, which fields start in the rows, which start across the top, what's totalled, what's hidden.

That "everything else loads with it" is the whole job. Eighteen analyses, one screen.

## Same Trick As Last Time

If you read the last post this shape will look familiar. One screen that doesn't change, and the bit that varies pushed out behind an interface:

```csharp
public interface IPivotHelper
{
    void LoadDataForPivot(DataArgs dataArgs, BusinessArgs businessArgs);
    IEnumerable<string> DefaultRowPropertyNames { get; }
    IEnumerable<string> DefaultColumnPropertyNames { get; }
    string DefaultDataPropertyName { get; }
}
```

That's Strategy again. The screen knows how to drive a pivot grid; each helper knows what one particular analysis looks like. Note the word *default* in those property names — they're only the starting position. Once the screen has loaded, the user drags things around and it's theirs.

The difference from last time is how the right one gets picked. The find screens used keyed dependency injection. This is an older desktop app and there was no container in it at all — I added one much later, and this was some of the first work I did there. So the choosing is done by hand, in a class I called a factory.

## The Factory

```csharp
public class PivotHelperFactory
{
    public static IPivotHelper GetPivotHelper(PivotQueryType queryType, PivotGridControl pivotGrid)
    {
        switch (queryType)
        {
            case PivotQueryType.SalesByRegion:
                return new SalesByRegionHelper(pivotGrid);
            case PivotQueryType.SalesByProduct:
                return new SalesByProductHelper(pivotGrid);
            case PivotQueryType.StockMovements:
                return new StockMovementsHelper(pivotGrid);
            // ...fifteen more
            default:
                throw new NotImplementedException();
        }
    }
}
```

That's it. The drop-down gives you an enum value, the factory hands back an `IPivotHelper`, and the screen never learns which class it got.

(One thing I'd change, fifteen years on: that `default` throws the wrong exception. Nothing is unimplemented — the caller has passed a value I don't handle, which is what `ArgumentOutOfRangeException` is for. It's a small thing, but the wrong exception sends whoever hits it hunting for missing code that was never missing. My own code, so I can be rude about it.)

There's a second reason this isn't done with dependency injection, and it's a better one than "there was no container". Look at the second parameter. The helper needs the live grid control off the screen — a real object that exists only once the form is up. A container can't hand you that; it doesn't know about it. Whenever the thing you're creating needs something only the caller has, a plain factory method beats a container, even in a modern app that's full of DI.

## So Is It Actually a Factory?

Here's the bit that had me unsure for years.

The Gang of Four book has two patterns with "factory" in the name, and this is neither of them.

**Factory Method** is where a base class has a creation method and *subclasses override it* to decide what gets made. The choosing is done by which subclass you're in, not by inspecting a value.

**Abstract Factory** is an object with several creation methods that produce a family of related things, where you swap the entire factory to swap the whole family.

A switch on an enum that returns one of eighteen implementations is neither. It never made the book. It picked up the name **Simple Factory** afterwards, because developers kept writing it regardless of what the catalogue said, and it needed calling something.

So I'd been comparing my code against two definitions that don't describe it, and quietly assuming I'd got it wrong. I hadn't. I'd written the version that turns up in real code far more often than either of the ones that got the official names.

The lesson I'd take from that isn't about factories. It's that "is this really pattern X?" is usually the least interesting question you can ask. The useful question is whether the thing hides a decision that would otherwise be splattered across the codebase. Mine does. Everything else is filing.

## What the Real One Looks Like

There's an Abstract Factory in another app I work on, and it's worth a look because it makes the difference obvious.

We call another company's API, and they were moving from one version of it to the next. Both had to work at the same time — the old one carrying the live traffic, the new one ready to take over on the day we switched. Two versions, both supported, and every call has to go out in the shape its version expects.

Put the two interfaces side by side and the difference is easier to see than any definition makes it sound:

```csharp
public interface IPivotHelper                           // nothing here creates anything
{
    void LoadDataForPivot(...);                         // does work
    IEnumerable<string> DefaultRowPropertyNames { get; } // describes itself
    IEnumerable<long> ExtractIdsForDrillDown();         // does work
}

public interface IPartnerApiFactory                     // every member creates something
{
    IPartnerApiClient GetClient(string url);            // makes a thing
    OrderRequest CreateOrderRequest(...);               // makes a thing
    AvailabilityRequest CreateAvailabilityRequest(...); // makes a thing
}
```

The pivot factory hands back something that does a job — it fetches data, it changes what's on screen, it answers questions about itself. The API factory hands back something with no behaviour of its own whatsoever. Every method on it manufactures an object you then go and use somewhere else.

And that's why there are several methods instead of one. There are two implementations, `PartnerApiV1Factory` and `PartnerApiV2Factory`, and each makes a matched set: ask the first for a client and you get the one that talks to the old endpoint, plus request objects shaped the way the old API expects. Ask the second and you get the new client and its own requests. A request only means anything to the client it was built for, so the three pieces have to agree — and choosing the factory chooses all of them at once.

On a migration that's the entire point. When two versions are live side by side, sending yesterday's request to today's endpoint is exactly the bug you're trying not to write, and this makes it difficult to do by accident.

As for which factory you get: a little switch on a configuration setting, no more interesting than the pivot one. Which is what made the switchover a non-event — no deployment, no code change, just flip the setting and every request from that moment builds itself the new way, with the old way one edit away if it misbehaved.

## The Same Idea, Grown Up

For a comparison, here's how the same job looks in a modern service I work on. Webhooks arrive as JSON from a third party, each one carrying an object type, and each type needs a different handler.

The registration builds a dictionary instead of a switch:

```csharp
services.RegisterHandler<CustomerWebHookHandler>(ObjectType.Customer);
services.RegisterHandler<OrderWebHookHandler>(ObjectType.Order);
```

And the factory looks the type up and asks the container for it:

```csharp
public IWebHookHandler Resolve(ObjectType objectType)
{
    if (_types.TryGetValue(objectType.ToString(), out var type))
    {
        return (IWebHookHandler)_provider.GetRequiredService(type);
    }
    // ...
}
```

Different plumbing, same pattern. The mapping lives in a dictionary rather than a `switch`, instances come from the container rather than `new`, and the key arrives in a JSON payload rather than from a drop-down. Key in, interface out, concrete type hidden. That's a factory in both cases, and realising the two were the same thing wearing different clothes was worth more to me than any definition.

## What It Bought

The pay-off shows up in what a single analysis costs to write. Here's a whole one:

```csharp
public class SalesByRegionHelper : AbstractSalesHelper, IPivotHelper
{
    public SalesByRegionHelper(PivotGridControl pivotGrid) : base(pivotGrid) { }

    public IEnumerable<string> DefaultRowPropertyNames
    {
        get { yield return nameof(SalesDataDTO.Region); }
    }

    public IEnumerable<string> DefaultColumnPropertyNames
    {
        get { yield return nameof(SalesDataDTO.Product); }
    }

    public string DefaultDataPropertyName => nameof(SalesDataDTO.Total);

    public override IEnumerable<string> HiddenPropertyNames
    {
        get
        {
            yield return nameof(SalesDataDTO.RecordId);
            yield return nameof(SalesDataDTO.DayOfWeek);
        }
    }
}
```

That's not really code so much as a description. Rows, columns, the number in the middle, and two fields the user doesn't need to see. Everything that actually does something — fetching the data, wiring the grid up, handling the drill-down back to the underlying records — lives further up in a base class shared by every analysis of that kind.

Those base classes weren't designed up front. They grew. I'd write two analyses over the same sort of data, notice the same code in both, and pull it up into a shared parent. Do that a few times and you end up with a small family of base classes, one per kind of data, each with a handful of concrete analyses under it. I'll come back to that in a later post, because it has a name too.

What a nineteenth analysis costs today depends on the data. If it's another view of something the app already understands, it's a class like the one above and one more line in the switch — half an hour, most of it spent deciding which fields should start where. If it needs a data source we've never pulled before, there's a new base class to write as well, because something has to fetch and shape that data before any analysis can sit on top of it. Cheap when it's a new view, a proper piece of work when it's new data.

## In Short

A factory is a class whose job is to decide which thing you get, so nothing else has to. Mine switches on an enum and news up a class; the modern one looks a type up in a dictionary and asks the container. Same pattern, different decade.

And if you've ever written one and wondered whether it counts as the real thing, it probably does. The version nearly everybody writes isn't in the book at all, which is a good reminder that the names came after the code, not the other way round.

This is the plain-English version — there's more to it, and Factory Method in particular is worth knowing properly if you're building a framework rather than using one. But this should cover the basics.
