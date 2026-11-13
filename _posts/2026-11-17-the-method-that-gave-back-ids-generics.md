---
layout: post
title: "The Method That Gave Back Ids: Generics and the Type You Threw Away"
pillar: patterns
series: design-patterns
excerpt: "A filter that was slow on a million-row test set. The filter method took a base class, returned a list of ids, and left the caller to loop through its original list matching them up. The type had been thrown away on the way in, so it could not come back out, and the matching was the cost."
tags:
  - design-patterns
  - csharp
  - dotnet
---

[Last time]({% post_url 2026-11-12-two-tables-one-query-program-to-the-interface %}) two tables got one interface and twenty queries stopped being written twice. That was the database side. The same events, once loaded, went through a second round of filtering in memory, and that is where this one starts.

It did not start as a tidy-up. The code worked, and working code gets left alone. The problem was speed. Every screen in the application shared one filter sidebar, and whatever you had loaded, the main event display, a report, an analytics view, those filters were applied to it. We had a test set of a million events, and on that set applying a filter was slow everywhere. The question was where the time went.

## A dozen shapes of the same thing

Every screen had its own DTO. The main event list, each of the reports, each of the analytics views, a dozen or so of them, each with the columns that screen needed and nothing else. They all derived from a base class, `BaseEventDto`, which held the handful of properties every one of them had: the event id, the start time, the deleted flag, the identities involved.

The data for a screen was loaded once, with the query from last time, and the filters in the sidebar were then applied in memory. That was deliberate: an analyst could change a filter and see the result immediately, without a round trip to the database for every tweak. So the filters ran over the loaded list. Hide deleted events. Show only events with confident locations. Collapse duplicates within a tolerance. Remove events from one source that were already represented by a record from another. Each of those is a pass over a list, and every screen needed them, so they lived in one service.

Which raises the question the service had to answer. It had to filter a `List<ReportEventDto>` for one screen and a `List<ListEventDto>` for another. What type does the parameter have?

## The answer that was there

The base class. This is reconstructed, because the original is gone, but the shape is the one I inherited:

```csharp
public List<long> GetFilteredEventIds(FilterArgs args, List<BaseEventDto> events)
{
    IEnumerable<BaseEventDto> result = events;

    if (args.DataDisplayArgs.IsHideDeleted)
    {
        result = result.Where(x => !x.Deleted);
    }
    if (args.DataDisplayArgs.IsShowConfidentLocationsOnly)
    {
        result = result.Where(x => x.HasConfidentLocation == true);
    }
    // ...the rest of the filters...

    return result.Select(x => x.EventId).ToList();
}
```

And every caller, in every screen's service, did this with it:

```csharp
var ids = _eventService.GetFilteredEventIds(args, events.Cast<BaseEventDto>().ToList());

var filtered = new List<ReportEventDto>();
foreach (var evt in events)
{
    if (ids.Contains(evt.EventId))
    {
        filtered.Add(evt);
    }
}
```

Read the two halves together and the logic is clear enough. The caller has a list of the concrete type. It casts it up to the base class so the shared method will accept it. The shared method filters and, because it only knows about the base class, it cannot give back the concrete list the caller started with. So it gives back the one thing it can, the surviving ids, and the caller walks its own original list picking out the rows whose ids survived.

## Where the time went

Count what that does on the million-row test set.

The cast copies every event into a new list. A million casts and a million-entry list, before any filtering has happened.

The filter returns the surviving ids. If the user's filter removes little, which is the common case, that is up to a million ids in a second list.

Then the loop. `ids.Contains(evt.EventId)` is a linear scan. `List<long>` has no index on its contents, so `Contains` walks the list from the start until it finds a match or runs out. For every one of the million events, it walks up to a million ids. If nothing was filtered out, that is a million times a million comparisons. A trillion, every time the user touches a filter.

That is the whole cost, and none of it is the filtering. The filtering is a handful of `Where` clauses that run once over the list. The cost is the join by hand on the way back out, done to recover a type that was thrown away on the way in.

## Why the type was lost

The cast is the problem, and it is worth being precise about why, because the obvious fix does not compile.

The method takes `List<BaseEventDto>`. You cannot pass a `List<ReportEventDto>` to it, because `List<T>` is invariant: a list of the derived type is not a list of the base type, since you could add a different derived type to it through the base reference. So the caller has to make a new list of the base type, which is what the `Cast` and `ToList` do. Variance is worth a post of its own, and it is on the list.

Now the method has a list of the base type. It can filter it. But it cannot return `List<ReportEventDto>`, because it does not know that is what it was given, and it cannot return `List<BaseEventDto>` usefully either, because the caller would have to cast every element back down. Returning ids sidesteps both. It is the natural thing to write once the cast is in, and the cast is the natural thing to write once the parameter is the base class.

So the real decision was the parameter type. Twelve DTOs, one method, the base class is what they have in common, and if generics are not a tool you reach for, the base class is the only way of saying that. It is just the wrong one.

## The type that goes in comes back out

The right tool is a type parameter with a constraint:

```csharp
public List<T> GetFilteredEvents<T>(FilterArgs args, List<T> events)
    where T : class, IFilterableEvent
{
    IEnumerable<T> result = events;

    if (args.DataDisplayArgs.IsHideDeleted)
    {
        result = result.Where(x => !x.Deleted);
    }
    if (args.DataDisplayArgs.IsShowConfidentLocationsOnly)
    {
        result = result.Where(x => x.HasConfidentLocation == true);
    }
    // ...the rest of the filters, unchanged...

    return result.ToList();
}
```

`T` is whatever the caller passed. The constraint says it must implement `IFilterableEvent`, which is the base class's shared properties pulled out into an interface, so the method can read `Deleted` and `EventId` and the rest without knowing anything else. And because `T` is the caller's type, the method can hand back a `List<T>` and the caller gets its own type back, filtered, with no cast and no second list:

```csharp
var filtered = _eventService.GetFilteredEvents(args, events);
```

One line. The cast is gone, so the million-entry copy is gone. The id list is gone. The loop is gone, so the trillion comparisons are gone. What is left is the filters, which run once over the list, and the list they return is the answer. On the million-row set that is the difference between a trillion comparisons and a million.

The constraint is an interface rather than the base class for the same reason as the previous post: it says what the method needs rather than where the type came from. The base class still exists and still implements the interface, so every existing DTO qualified without changing. But a type that is not a `BaseEventDto` can qualify too, and one does.

## What came with it

I went in for the speed. The readability came free, and it is worth saying what it bought, because it is the reason the fix stayed fixed.

Every caller lost a loop and a cast. The method's signature now says what it does: give me a list of things I can filter, and I will give you the same list back, filtered. Nobody reading it has to work out that the `List<long>` coming back is meant to be matched against something. And the compiler knows `filtered` is a `List<ReportEventDto>` without being told, so the next method along gets the real type and does not need its own cast.

None of that would have justified touching working code on its own. It is what you get for having to touch it anyway.

## What was left behind

Not everything went. The old id-returning method is still on the service interface, now generic, next to the version that returns the list. Nothing much calls it. And the id-list-plus-loop idiom survives elsewhere in the application, in places I never got to, doing the same scan over the same sizes of data. Fixing the shape in one path does not fix the habit.

## In short

If a shared method takes a base class and returns ids, it is because the type was thrown away on the way in and there was no way to give it back. The caller is then left rebuilding its own list by hand, and that rebuild is a scan inside a loop. On a million rows it is a trillion comparisons, and that is where the time goes.

A type parameter constrained on an interface keeps the caller's type all the way through. The method says what it needs, works on anything that provides it, and returns what it was given. The copy, the id list and the loop all disappear, and the readability comes along for nothing.
