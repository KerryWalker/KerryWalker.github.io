---
layout: post
title: Lists, LINQ Where and Dictionaries — .NET Performance in Plain English
excerpt: Why filtering one list against another grinds to a halt on real-world data, and how a HashSet or Dictionary fixes it — explained with a child's birthday party. First in a short series on .NET performance problems.
tags:
  - csharp
  - dotnet
  - performance
  - linq
---

One of the things I come across a lot in code reviews and performance problems is lists — and often multiple lists — being used where a dictionary would be a far better collection. It's usually caused by not understanding the difference between finding records in a list versus a dictionary or HashSet, and what's commonly known as Big O notation. That sounds like computer-science theory, but by the end of this post you'll have the whole idea from nothing more complicated than a child's birthday party.

This is the first in a short series about performance problems I keep meeting in .NET code, explained in plain English. Here's my real-world example.

## The Real-World Example

When code needs to filter a large amount of data, I often find LINQ doing the work:

```csharp
var activeCases = caseList.Where(x => x.Status == status);
```

This is fine. Even with a large amount of data it's a single pass through `caseList` — one look at each item.

The problem starts when there's a second collection involved. Say we have a list of case ids and we want the matching cases:

```csharp
var cases = caseList.Where(x => caseIdList.Contains(x.CaseId));
```

Still one line, still reads fine. But strip away the extension methods and this is roughly what runs:

```csharp
foreach (var item in caseList)
{
    foreach (var id in caseIdList)
    {
        if (item.CaseId == id)
        {
            yield return item;
            break;
        }
    }
}
```

A loop inside a loop. `Contains` on a `List` has no shortcut. It starts at the front and checks every element until it finds a match or runs out, so for every case we potentially read the entire id list. You get the same thing with `caseIdList.Any(y => y == x.CaseId)`. It's the same work under a different method name.

To see why that matters, forget the code for a minute.

## The Birthday Party

Imagine you're organising a birthday party for one of your children. They've given you a list of 10 names, and the road the guests live on, which has 100 houses in it.

You go to the first house, knock on the door, and ask whether guest 1 lives there. Whether they do or not, you then go next door and ask about guest 1 again, repeating for every house in the street. When you get to the final house, you go back to the first house and start again with guest 2. And so on for every name on the list.

10 guests, 100 houses: 1,000 door-knocks. Not a lot on modern hardware, and with the small numbers we use as developers everything feels instant. But out in the real world someone has a million rows, and the software gets used in a way where you're checking a million houses for a million guests. Now it's 1,000,000,000,000 operations — a trillion knocks.

**Flipping the lists doesn't help.** You might think swapping which collection you loop over would fix it — knock on each door once and ask about every guest while you're there. You'd walk less, but you still ask the same total number of questions: 100 houses × 10 guests is still 1,000. In code, swapping the lists changes the shape of the loops, not the amount of work.

## Fix One: Learn the Guest List (HashSet)

The first fix comes from noticing what the question actually is. At each door, all you want to know is: *is this child on the guest list, yes or no?*

So learn the list. Walk the street once, and at each door ask the child's name and check it against the names you're holding in your head — one instant check per house. That's 100 doors plus the 10 names you memorised up front: about 110 operations instead of 1,000.

In C#, "learning the list" is a `HashSet`:

```csharp
var caseIds = caseIdList.ToHashSet();

var cases = caseList.Where(x => caseIds.Contains(x.CaseId));
```

The filtering line barely changes — it's still `Where` and `Contains`. But `Contains` on a `HashSet` doesn't scan; it computes where the value *would* be and looks in exactly that spot. One check, no matter whether the set holds ten ids or ten million.

Note which collection changed. `caseList` stays a list — we're walking through it, not searching it. Only the collection on the receiving end of `.Contains()` needs to become the fast one.

## Fix Two: House Numbers on the List (Dictionary)

Now imagine the guest list your child handed you had the house numbers written next to the names. You don't knock on any doors at all — you go straight to number 42, deliver the invitation, and move on. 10 direct visits instead of 1,000 knocks.

That's a `Dictionary`: not just "is this name on the list?" but "take this name and hand me back the thing that goes with it."

```csharp
var casesById = caseList.ToDictionary(x => x.CaseId);

var cases = caseIdList.Select(id => casesById[id]);
```

The difference between the two fixes is what you get back. A `HashSet` answers yes/no — did this case survive the filter? A `Dictionary` fetches the value — give me the case for this id. If you catch yourself checking membership with a `HashSet` and then searching the list anyway to get the object, you wanted a `Dictionary` all along.

## Building It Isn't Free

Neither of these comes for free. `ToHashSet` reads every id once, and `ToDictionary` walks the whole street once, knocking on every door and writing down who lives where. That can feel like a waste. You've visited all 100 houses to build a directory you'll only use 10 times.

But go back to the million-row case. Building the dictionary is a million operations, and the million lookups are a million more: two million in total, against the trillion we started with. And that's all Big O notation really is, the thing you hear about all the time but never quite pin down. The nested loops are O(n×m), which just means the work is the two sizes multiplied together. A hash lookup is O(1), a fixed cost no matter the size.

## Where It Gets More Complicated

There are times when it isn't worth it: a small list, a single lookup. If you know your list is going to be small, say you're checking one id against twenty items, then building anything is a waste. These structures earn their keep when you look things up over and over, and that's a call you need to make when you build the system.

There's a lot more to it than this. In the real world a single property is rarely enough to identify a record, so you often need a key made up of several properties at once. That means choosing between a value tuple for the key or writing your own `IEqualityComparer`, and getting it wrong can lose you the speed you came for, or match records that shouldn't match. But this should cover the basics.

