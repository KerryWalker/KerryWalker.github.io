---
layout: post
published: false
title: A Builder That Stores Questions, Not Answers
pillar: patterns
series: design-patterns
excerpt: "Declaring a menu where every entry might be hidden by a permission, a config setting or a feature flag. A fluent builder, and the decision that matters more than the chaining: storing how to find out rather than what the answer was."
tags:
  - design-patterns
  - csharp
  - dotnet
---

The main menu in our app is a decent size. Thirty-odd categories down the side, options under each, well over a hundred entries once you count them.

Hardly any of it is the same for two customers. Some options need a permission. Some only appear once a bit of configuration is filled in. Some sit behind a feature flag while they're being built. So the menu isn't a list, it's a list that has to work out what it's allowed to be.

This post is about how we declare one. The next is about the bit I never wrote: why a category you have no permission to open disappears on its own.

## Declaring a category

Here's a whole one:

```csharp
var find = new MainMenuItem(labels.Find, context)
    .WithIcon("material-symbols-outlined findIcon");

find.AddSubItem(labels.Orders)
    .WithVisibilityRule(() => user.HasPermission(Permissions.FindOrders))
    .WithNavigateUrl(Pages.Orders.FindOrder);

find.AddSubItem(labels.Quotes)
    .WithVisibilityRule(() => user.HasPermission(Permissions.FindQuotes))
    .WithNavigateUrl(Pages.Quotes.QuoteSearch);

find.AddSubItem(labels.Jobs)
    .WithVisibilityRule(() => user.HasPermission(Permissions.FindJobs))
    .WithFeatureFlag(FeatureFlag.Create(Entries.EnableJobs), featureFlagService)
    .WithNavigateUrl(Pages.Jobs.FindJob);
```

Every `With` method returns the item, so the settings chain. One statement per entry, the rule sitting next to the thing it applies to, and nobody has to open a second file to work out why an option isn't showing.

That's the **Builder** pattern, and on its own it's not very interesting. Chaining reads nicely. So what.

The bit worth talking about is what those methods actually put in the object.

## Storing the question

Look at the visibility rule again:

```csharp
.WithVisibilityRule(() => user.HasPermission(Permissions.FindOrders))
```

That isn't a `bool`. It's a lambda, and the builder keeps it as one:

```csharp
public Func<bool> VisibleRule { get; private set; }

public MainMenuItem WithVisibilityRule(Func<bool> visibilityRule)
{
    VisibleRule = visibilityRule;
    return this;
}
```

So the menu item doesn't know whether you can see it. It knows **how to find out**, and it finds out every time somebody asks:

```csharp
if (VisibleRule is not null && !VisibleRule.Invoke())
{
    return false;
}
```

That distinction is the whole design. A builder that evaluated `user.HasPermission(...)` at the point of construction would produce a menu that was correct at the moment it was built and slowly wrong afterwards. Permissions change. Configuration changes. Somebody toggles a feature flag. Build once, store the answer, and the menu is a photograph of a situation that has since moved on.

Storing the question means the menu is never stale. There's nothing to refresh, nothing to invalidate, no event to subscribe to. The answer is computed at the moment of asking, which is the only moment it's guaranteed to be right.

## The cost of doing it that way

It isn't free, and I'd be glossing over things if I said otherwise.

Those rules run constantly. Every render, for every item in the menu, each lambda is invoked. That's fine when a rule is a dictionary lookup against the user's permissions. It is decidedly less fine if somebody writes a rule that goes and asks the database something, because nothing in the signature stops them, and `Func<bool>` looks equally cheap whatever is hiding inside it.

We have one of those, near enough. The feature flag check happens inside the same property, synchronously, by blocking on an async call. It works, it has never caused us a problem, and I would not write it that way again.

The other cost is that a closure captures things. `() => user.HasPermission(...)` holds on to `user` for as long as the menu exists. That's what makes it work, and it's also how you'd accidentally keep something alive far longer than you meant to.

## One more detail

`AddSubItem` does something small that makes the whole thing read better:

```csharp
public MainMenuItem AddSubItem(string title, bool addToBottom = false)
{
    var newMenu = new MainMenuItem(title, _context);
    if (!addToBottom)
    {
        int x = _items.ToList().BinarySearch(newMenu, new MainMenuItem_Text_Comparer());
        _items.Insert((x >= 0) ? x : ~x, newMenu);
    }
    else
    {
        _items.Add(newMenu);
    }
    return newMenu;
}
```

It returns **the child**, not the parent. So the chain that follows configures the item you've just added rather than the one you called it on.

It also drops the new entry into its alphabetical place on the way past. Add an option two years later and it lands where it belongs, which is the sort of thing you only appreciate when you've seen the alternative.

## Where you've already met this

Any builder that takes a delegate rather than a value is doing the same thing. `services.AddScoped<IThing>(sp => new Thing(...))` hands the container a recipe instead of an object, because the container needs to make a new one per scope, not hand out the one you made at startup.

Validation libraries do it too. `RuleFor(x => x.Name).Must(BeUnique)` stores the check, not the result, because the result depends on which object you're validating.

Any time you've passed a lambda into a configuration method, somebody decided the answer wasn't knowable yet.

## In short

When you're building something whose truth changes after you've finished building it, store the question rather than the answer. The object stays correct without anyone having to remember to update it.

Which leaves one thing unexplained. Those rules are all on the individual options. Nothing above says what happens to the **Find** category itself when a user has no permission for anything inside it, and yet it disappears. Next time.
