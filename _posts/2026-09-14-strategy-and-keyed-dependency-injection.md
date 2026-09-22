---
layout: post
title: "One Screen, Many Searches: Strategy and Keyed DI"
pillar: patterns
series: design-patterns
excerpt: A single search screen that works for customers, invoices, orders and every other entity in the app. One interface, many behaviours, and the keyed dependency injection that picks the right one. The Strategy pattern, in real code.
tags:
  - design-patterns
  - csharp
  - dotnet
---

At the end of [the first post in this series]({% post_url 2026-09-09-youre-already-using-design-patterns %}) I left a question hanging. I'd shown a single search screen handed a different view model on every page (the Strategy pattern, whether you call it that or not) and said that if you were wondering how each page gets the right one, that was a post of its own. This is that post.

Fair warning: this one goes a little deeper than the rest of the series. The patterns themselves are things you're already doing by accident. Picking one implementation out of many is something you go looking for on purpose. Normal service resumes next time.

## The Problem

Every business app has search screens, and lots of them. Find a customer, find an invoice, find an order. On the surface they all look the same: a set of search boxes, a results grid, paging, sorting, columns you can show and hide, layouts you can save. Only the details change: what you search on, what columns come back, where the data comes from.

An earlier version had a single screen doing the rendering, which was the right instinct, but every entity had its own interface:

```csharp
// A separate interface for every screen
public class FindCustomersViewModel : ViewModelBase, IFindCustomersViewModel { }
public class FindInvoicesViewModel : ViewModelBase, IFindInvoicesViewModel { }
// ...and so on for every screen

// So each page had to inject its own specific type
[Inject] public IFindCustomersViewModel ViewModel { get; set; }
```

Adding a new search screen meant a new interface, a new implementation, and a fair bit of plumbing in a shared base to line them all up. It worked, but it was more ceremony than it needed to be, and the wiring was the kind of thing you got wrong at four o'clock on a Friday.

## The Strategy

The shape I wanted was one screen and one interface, full stop.

The screen knows how to render a search: boxes, grid, paging, saved layouts. It knows nothing about customers or invoices. Everything entity-specific sits behind a single interface:

```csharp
public interface IFindViewModel
{
    IReadOnlyList<Column> Columns { get; }
    IReadOnlyList<SearchField> SearchFields { get; }
    Task GetData();
}
```

There's a `FindCustomersViewModel`, a `FindInvoicesViewModel`, an orders one, and a load more. Each one decides what "find" means for its entity. In our case the columns come straight from the stored procedure that fetches the data, so a new screen barely has to describe itself at all. The screen takes an `IFindViewModel` and talks to nothing else. The strategy decides *what* to find; the screen decides *how* to show it.

That's textbook Strategy: a stable thing (the screen) that delegates the bit that varies to an interchangeable implementation. If you've read the first post this is familiar. The interesting part is the next question.

## How the Right One Gets Picked

The screen takes an `IFindViewModel`. But there are 12 of them, all registered against that one interface. When the customers page loads, how does it get the customers one and not the invoices one?

The answer is **keyed dependency injection**. Instead of one implementation per interface, you register many, each under a key:

```csharp
services.AddKeyedScoped<IFindViewModel, FindCustomersViewModel>(nameof(FindCustomersViewModel));
services.AddKeyedScoped<IFindViewModel, FindInvoicesViewModel>(nameof(FindInvoicesViewModel));
// ...and so on
```

And each page asks for the one it wants by naming that key:

```csharp
[Inject(Key = nameof(FindCustomersViewModel))]
public IFindViewModel ViewModel { get; set; }
```

Using `nameof(...)` means the key is just the class's own name, so there's nothing to keep in sync by hand. The customers page asks for the customers one, the screen renders it, and neither of them knows the other exists. No concrete types leaking into the page, no big switch statement deciding who gets what, no loader to wire up. A new screen is a new view model and one line.

(In our case a home-grown attribute does the registering, but it's the framework's keyed services doing the work underneath.)

## Where You've Already Met This

Keyed services have been built into .NET since version 8: `AddKeyedScoped` and friends to register, `[FromKeyedServices("key")]` or `[Inject(Key = ...)]` to resolve.

And the plainer version, from the first post: any time you inject an interface so you can swap the implementation, that's Strategy. Keyed DI is just Strategy for when there are lots of implementations and something has to pick.

## The Trade-off

The keys are strings under the covers. `nameof` takes most of the sting out (rename the class in your IDE and the key comes with it), but it's still a string at runtime, so a typo or a hand-written key won't fail until something asks for a service that was never registered. It's a trade: you lose a little compile-time safety, and in return the page no longer has to know a single concrete type. For this, with every screen behaving the same way, that trade is well worth it. For two implementations, I'd probably just inject the one I wanted.

## The Payoff

Adding a new search screen used to mean a new interface, a new implementation, and wiring it into the loader without breaking the others. Now it's a view model and a one-line registration, and the screen it plugs into hasn't changed since the day I wrote it. The duplication that used to spread with every new entity just doesn't happen any more.

That's what a pattern buys you when it fits: not cleverness for its own sake, but the boring, valuable result of writing the hard part once.

This is the plain-English version, of course. There's more to keyed DI: service lifetimes, what happens when a key isn't found, resolving a whole set of keyed services at once. But this should cover the basics.
