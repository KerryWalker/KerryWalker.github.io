---
layout: post
title: "The Pattern the Language Ate: Iterator"
pillar: patterns
series: design-patterns
excerpt: "Across two codebases and well over a hundred projects I can find one iterator written by hand, and it is a wrapper round List. The pattern is everywhere. It is just that the compiler writes it now, and the one time I tried to take it back, a grid control took it off me."
tags:
  - design-patterns
  - csharp
  - dotnet
---

Every pattern in this series so far has been something I could point at in a codebase and say "there, that class". This one is different. It is in every codebase I have ever worked on, thousands of times over, and I cannot find one that anybody wrote.

That is not a failure of the pattern. It is what success looks like.

## What it was for

Iterator is the idea that walking through a collection should be separate from the collection itself. The caller asks for the next thing, and the next, and does not know or care whether the things live in an array, a linked list, a tree, or are being made up on the spot. The collection decides how to traverse itself. The caller just says "next".

In 1994 that needed explaining, because the alternative was a `for` loop with an index, and an index only works on things that have one. By the time C# arrived it had a keyword. `foreach` asks for an enumerator and calls `MoveNext` until it says no. In C# 2 it got `yield return`, and from then on the compiler would write the enumerator for you from a method that looked like ordinary code.

So the pattern did not go away. It got absorbed. Which raises the question of what is left to say about it, and the answer is in what happens when you go looking.

## Going looking

Two codebases. The one I work on now, and the one from the job before, which is around sixty-five projects.

In the old one there are around eighty files that use `yield return`. I can find exactly one that mentions `IEnumerator<T>` at all, and nothing that implements `GetEnumerator` by hand to walk a structure of its own.

In the current one it is a dozen or so files with `yield return` and a handful that mention the enumerator interface. Of those, one is the textbook case: a thirty-line class that enumerates the values of an enum type, written when somebody wanted to `foreach` over one. It is correct, it is the whole pattern in one screen, and it was written once.

Everything else is `yield`. Here is the one of mine I would keep:

```csharp
private IEnumerable<string> GetMessageForValues(string measurement, PricingRule rule)
{
    var fromNull = rule.IsNull($"{measurement}_FROM");
    var toNull   = rule.IsNull($"{measurement}_TO");
    var from     = rule[$"{measurement}_FROM"];
    var to       = rule[$"{measurement}_TO"];

    if (!fromNull)
    {
        if (!toNull)
        {
            yield return $"{measurement} is between {from} and {to}";
        }
        else
        {
            yield return $"{measurement} is greater than {from}";
        }
    }
    if (fromNull && !toNull)
    {
        yield return $"{measurement} is less than {to}";
    }
}
```

It builds the wording that explains why a pricing rule applied. A rule can have a lower bound, an upper bound, both or neither, for each of a few measurements of the item, and the message the user sees has to say which. The caller does this:

```csharp
ruleSettings.AddRange(GetMessageForValues("length", rule));
ruleSettings.AddRange(GetMessageForValues("width", rule));
ruleSettings.AddRange(GetMessageForValues("age", rule));
```

Nothing about that is clever. The point is what it replaced. Without `yield` the method is the same `if` tree with a `List<string>` declared at the top, an `Add` inside every branch, and a `return` at the bottom. Three lines of ceremony around the logic, and the ceremony is where the bugs go. The branch that forgot to add. The early `return` someone drops into the middle of the method without reading to the bottom, which skips the list entirely and sends an empty one up the stack, and I have lost hours further up that stack before finding it. With `yield`, the method produces results the way it thinks about them, one at a time, and the collecting happens somewhere else. There is no list to skip. An early `return` just ends the sequence, and everything produced before it has already been handed over.

That is Iterator doing its job. The traversal is separate from the thing being traversed, and here the thing being traversed does not even exist as a collection. The sequence is the method.

## The one I wrote by hand, and what happened to it

There is one place in the current codebase where I did implement the enumerator myself, and it is more instructive than anything that works.

We have a set of collection classes that carry change tracking. Add an item and it is marked as added. Remove one that came from the database and it is not removed, it is marked as deleted and kept, so the save knows to issue a delete. Every item knows its own state and the collection maintains it.

The first version was the proper shape for that. A class implementing `IList<T>`, holding a private `List<T>`, with `Add` and `Remove` and `Insert` doing the state bookkeeping before forwarding to the inner list, and `GetEnumerator` handing out the inner list's enumerator. A collection that owns its own rules, and the only way through it is the front door.

Then it was bound to a grid control, and the grid did not reload when its data source was an `IList<T>` rather than a `List<T>`. A bug at their end, with a ticket. Not something I could fix from my side.

So the class now inherits `List<T>` instead, and every method that used to forward is marked `new`:

```csharp
// This class has been changed to inherit from List<T> because implementing
// IList<T> was not behaving correctly with the grid. This will be changed
// back when the bug has been fixed.
public class RecordItems<T> : List<T>, IRecordItems where T : RecordItem
{
    // readonly IList<T> _list = new List<T>();  // UNCOMMENT WHEN CHANGING BACK

    public new void Add(T item)
    {
        if (item.DataState == DataRowState.Detached) item.DataState = DataRowState.Added;
        base.Add(item);
    }

    public new bool Remove(T item)
    {
        if (item.DataState == DataRowState.Added)
        {
            return base.Remove(item);
        }
        item.DataState = DataRowState.Deleted;
        return true;
    }

    // public IEnumerator<T> GetEnumerator()
    // {
    //     return _list.GetEnumerator();
    // }
}
```

The commented-out original is still in the file, with a note saying to put it back. It has been like that for three years.

What was lost is the thing the pattern was protecting. `new` hides a method rather than overriding it, so anything holding the object as a `List<T>` calls the base `Add` and the state tracking never runs. The front door is still there. The wall next to it is gone. The grid is happy, and the collection is now a `List` with suggestions attached.

I am not telling that story to complain about the grid. I am telling it because it answers the question this pattern always raises in a language that has `foreach`: why would anybody write one by hand? The answer is that you do it when you want a collection to own its own rules, and the thing to know before you start is that a lot of the framework and most third-party controls are written against `List<T>` and `IList<T>` and will not thank you for being anything cleverer. The custom collection is the right design and it lost anyway.

## Where you have already met this

Everywhere, which is the point. Every `foreach`, every LINQ operator, every `await foreach` over an `IAsyncEnumerable`. `Where` and `Select` are iterators that wrap other iterators, and the reason a LINQ chain does nothing until you enumerate it is that nothing has asked for the first element yet.

That laziness is the one bit of the pattern worth keeping an eye on, because it cuts both ways. A `yield` method that does real work per item, reflecting over properties or reading from a stream, is lazy in a way that matters: you can stop early and the rest never runs. A `yield` method that builds the whole list first and then yields each item is lazy in a way that does not, and there is one of those in the codebase too. It is not wrong. It is just an enumerator that costs the same as the list it is hiding.

## In short

Iterator is the pattern that got built into the language, and the evidence is that you cannot find one anybody wrote.

Use `yield` when a method produces a sequence, because it lets the logic read as logic rather than as list management. Write the enumerator yourself only when a collection needs to own its own rules, and expect the rest of the ecosystem to push back, because it was written for `List<T>` and it will treat anything else as a bug. Sometimes it will be their bug and you will still be the one who changes.
