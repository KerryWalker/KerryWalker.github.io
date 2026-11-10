---
layout: post
title: "Two Tables, One Query: The Principle Under the Patterns"
pillar: patterns
series: design-patterns
excerpt: "Every analytics query in the application was written twice, once per table, with an if statement choosing between the copies. The fix was not one of the twenty-three patterns. It was the sentence near the front of the book that all of them are built on."
tags:
  - design-patterns
  - csharp
  - dotnet
---

Every pattern in this series has been a named shape out of the book. This one is not. It is the sentence near the front of the book that the named shapes all depend on, and the reason I want to write it up is that I have a before and after for it that I think about more than any of the patterns.

*Program to an interface, not an implementation.* Everyone has read it, and plenty of experienced developers will tell you it is overrated. Interfaces complicate the code. You are never going to swap the implementation for a different one. If you are not injecting it, you do not need it. All of which misses what an interface is actually for. Swapping implementations and dependency injection are two things you can do with one. Removing duplication is another, and it is the one this post is about. Here is what the code looked like when it was not being done, and what changed when it was.

## Two tables that are the same table

A previous job: an application that analysed large sets of events, each one with a location. The locations came from the data as recorded, and they could also be corrected afterwards by an analyst who knew better. So there were two SQL views over the same events. One gave you the locations as recorded. The other gave you them as corrected. Same columns, around a hundred and fifty of them, same rows, different values in the location fields.

Which one you wanted was a setting on the screen. Every analytics query, and there were a lot of them, had to run against whichever the user had picked.

Entity Framework scaffolded them as two entity classes, `EventView` and `EventOverrideView`, each with the same hundred and fifty properties, each in its own `DbSet`. Two classes with no relationship to each other as far as the type system was concerned.

## What that did to the queries

The filtering was already pulled out into helpers, which was the right instinct. The trouble was that there were two sets of them. One static class for the recorded view, one for the corrected view, and every method in one had a twin in the other that differed only in its type:

```csharp
public static class EventViewFilter
{
    public static IQueryable<EventView> ApplyJobFilter(Guid jobId, IQueryable<EventView> events)
    {
        return events.Where(x => x.JobId == jobId.ToString());
    }

    public static IQueryable<EventView> ApplyDateFilter(DateTimeArgs args, IQueryable<EventView> events)
    {
        // ...
    }
    // ...and the rest
}

public static class EventOverrideViewFilter
{
    public static IQueryable<EventOverrideView> ApplyJobFilter(Guid jobId, IQueryable<EventOverrideView> events)
    {
        return events.Where(x => x.JobId == jobId.ToString());
    }

    public static IQueryable<EventOverrideView> ApplyDateFilter(DateTimeArgs args, IQueryable<EventOverrideView> events)
    {
        // ...identical
    }
    // ...and the rest, identical
}
```

Close to two hundred lines each, and the bodies are character for character the same. They had to be, because `IQueryable<EventView>` and `IQueryable<EventOverrideView>` are different types with no common ancestor, and a method that takes one cannot be handed the other.

Then every query had to pick a set. Reconstructed, because the originals are gone, but this is the shape:

```csharp
public List<CellUsageDto> GetCellUsage(DataArgs args)
{
    using (var context = GetContext())
    {
        if (args.LocationSource == LocationSource.Corrected)
        {
            var events = EventOverrideViewFilter.ApplyAllFilters(args, context.EventOverrideView);
            return events.Select(x => new CellUsageDto { ... }).ToList();
        }
        else
        {
            var events = EventViewFilter.ApplyAllFilters(args, context.EventView);
            return events.Select(x => new CellUsageDto { ... }).ToList();
        }
    }
}
```

Two branches that differ in a class name and a property name, then a projection written twice. Multiply that by every analytics query in the data layer.

The duplication was not a style problem. It was that any change to a filter had to be made in two classes, and the second one was the one that got missed. A fix applied to the recorded filter and not the corrected one is a bug that only shows up for users with a particular setting, and it looks exactly like a data problem when it is reported.

The duplication was there because the two classes had no type in common, and nothing about scaffolded code suggests they could have one. They can. The two things that make it possible are both ordinary C#, and the rest of this post is how to put them together.

## Scaffolded classes are partial

Every class EF scaffolds from a database is declared `partial`. That is there for exactly this reason: so you can add to the class in a file of your own, and the scaffolder can regenerate its file without touching yours. Most people only ever use it to add a method or two. It also lets you add an interface.

So the whole change on the entity side was two new files, one line each:

```csharp
public partial class EventView : IEventView { }
public partial class EventOverrideView : IEventView { }
```

And an interface declaring the properties the queries use. The properties already exist on both scaffolded classes with the same names and types, so the interface is satisfied the moment it is declared. In practice the queries used nearly all of them, so the interface is long, but it is a list of properties and nothing else.

Regenerate the entities tomorrow and the scaffolder rewrites its half of each class. The interface stays, because it is in your file.

## One place knows there are two tables

Then one method on the context, and it is the only place in the codebase that knows there are two tables:

```csharp
public IQueryable<IEventView> GetEvents(LocationSource? source)
{
    switch (source ?? LocationSource.Recorded)
    {
        case LocationSource.Recorded:
            return EventView.AsNoTracking();
        case LocationSource.Corrected:
            return EventOverrideView.AsNoTracking();
        default:
            throw new NotImplementedException();
    }
}
```

The `if` did not disappear. It moved, once, to the one place a decision about which table to use belongs. Everything downstream gets an `IQueryable<IEventView>` and does not know, and cannot find out, which table it came from.

The filter helpers became generic, constrained on the interface:

```csharp
public static IQueryable<T> ApplyJobFilter<T>(this IQueryable<T> events, Guid jobId)
    where T : IEventView
{
    return events.Where(x => x.JobId == jobId.ToString());
}

public static IQueryable<T> ApplyDateFilter<T>(this IQueryable<T> events, DateTimeArgs args)
    where T : IEventView
{
    if (args?.Range != null && args.IsActive)
    {
        events = events.Where(x => x.EventStart >= args.Range.From
                               && x.EventStart <= args.Range.To);
    }
    return events;
}
```

And the query from before became this:

```csharp
public List<CellUsageDto> GetCellUsage(DataArgs args)
{
    using (var context = GetContext())
    {
        var events = context.GetEvents(args.LocationSource)
                            .ApplyAllFilters(args, excludeDeleted: true);

        return events.Select(x => new CellUsageDto { ... }).ToList();
    }
}
```

One branch. Twenty-odd queries across nine data access classes all went the same way, and the per-table filter helpers were left with nothing calling them.

## Why the queryable can be typed as the interface

Two details make the rest of it work, and both are worth knowing because they come up again anywhere you do this.

`GetEvents` returns `IQueryable<IEventView>`, but what it actually hands back is a `DbSet<EventView>`. That compiles because `IQueryable<T>` is covariant: a queryable of the concrete type is also a queryable of anything the concrete type implements. You get that for free, the same way an `IEnumerable<string>` is an `IEnumerable<object>`.

The filter then does `x.JobId == jobId` where `x` is typed as the interface, and that lambda is not run, it is handed to Entity Framework as an expression tree to turn into SQL. The member in the tree is `IEventView.JobId`. EF resolves that to the implementing entity's property and translates it like any other column, so the generated SQL is identical to what the per-table version produced. The entire analytics layer ran that way in production, against both tables, with every filter written once.

## Why this is the principle and not a pattern

Look at what the change actually was. Nothing was wrapped. Nothing was injected. No factory, no strategy, no adapter. Two classes were given a common interface, and the code that used them was rewritten against the interface instead of the classes.

That is the sentence from the book, and it is what every one of the patterns is a particular arrangement of. Strategy is programming to an interface so you can swap the algorithm. Adapter is programming to an interface so you can swap the thing behind it. Proxy is programming to an interface so you cannot tell what is behind it. Strip the specific intent off any of them and this is what is left.

Which is why it is worth having an example that is just this. Two tables, one interface, one decision made once. The patterns are for when the problem has a more particular shape. Most of the time it does not, and this is enough.

## What it costs

The interface has to be kept in step with the views. Add a column to both views and forget the interface, and the column is there in both entity classes and invisible to every query, because the queries can only see the interface. That is the right failure, since it fails at compile time the moment a query tries to use it, but it is a third place to update.

## In short

Two tables with the same shape do not need two copies of every query. Give them an interface, make the choice of table in one place, and write everything else against the interface.

The decision does not go away. It moves to the one place it belongs, and the twenty places that used to make it stop having to. That is the whole of the principle, and most of the time it is all you need before reaching for a named pattern.

Later in the series, the other half of the same story. The code that filtered those events in memory had a different problem, and it was costing more than readability.
