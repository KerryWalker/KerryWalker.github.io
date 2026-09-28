---
layout: post
title: "The Menu That Hides Itself: The Composite Pattern"
pillar: patterns
series: design-patterns
excerpt: Every option in our menu declares its own permission rule. No category declares anything, and yet empty categories disappear on their own. That's the Composite pattern, and it's the reason nobody has to write the rule you'd expect to find.
tags:
  - design-patterns
  - csharp
  - dotnet
---

[Last time]({% post_url 2026-09-29-a-builder-that-stores-questions %}) I showed how we declare a menu category, and left one thing hanging.

Every option carries its own visibility rule. Find Orders needs a permission, Find Quotes needs a different one, Find Jobs needs a permission and a feature flag. All declared on the individual items.

Nothing is declared on the **Find** category itself. And yet if you're a user with none of those permissions, Find isn't there. Not empty: gone.

Go looking for the code that does that and you won't find it. There is no "hide a category when all its children are hidden" rule anywhere in the application.

## One type, two jobs

The reason is in the shape of the class. A menu item holds menu items:

```csharp
public class MainMenuItem
{
    private List<MainMenuItem> _items;
    public IList<MainMenuItem> SubItems => _items;

    // ...
}
```

There's no separate `Category` and `Option`. There's one type, and whether it behaves as a branch or a leaf depends entirely on whether anything has been added to it. Find is a `MainMenuItem` with children. Find Orders is a `MainMenuItem` without any.

That's the **Composite** pattern, and that one decision is what makes the disappearing work.

## Asking the tree

Here's the end of the item's `Visible`:

```csharp
// if its not got a url and its got no subitems or all the subitems are hidden, hide it
if (_items.Count == 0 || !_items.Select(i => i.Visible).Any(v => v))
{
    return false;
}

// just return true and if it shouldn't be showing figure out what I missed
return true;
```

A category doesn't decide whether it's visible. It asks its children, and each child answers the same way: check my own rule, check my own feature flag, then ask my own children.

So a single question at the top of the tree walks all the way down, and the answer comes back up. Take away someone's permission for Orders, Quotes and Jobs and each of those three answers "no" for its own reasons, Find gets three noes and concludes there's no point existing, and the category vanishes.

Nobody wrote that. It falls out of the fact that a branch and a leaf are the same type answering the same question.

## The same trick, a different question

Once the tree can answer one question it can answer others, and we use it for the "new" and "updated" badges:

```csharp
if (_items.Any() && _items.All(p => p.GetStatus() == MainMenuItemStatus.New))
{
    // the whole category is new
}

if (_items.Any(p => p.GetStatus() == MainMenuItemStatus.New
                 || p.GetStatus() == MainMenuItemStatus.Updated))
{
    // something in here has changed
}
```

Different question, same walk. If everything under a category is new, the category is new. If anything under it has changed, it's flagged as updated. Two lines of aggregation rather than any bookkeeping about which categories contain what.

The first recursive question is a bit of work. Every one after that is nearly free, because the structure was already doing the walking.

## Where you've already met this

Folders. Asking a folder its size asks everything inside it, and those are folders too. It's the reason the answer takes a while and the reason nobody had to write special code for nested folders.

The DOM. A `<div>` holds elements and one of them can be another `<div>`. Hide the parent and everything inside goes with it, because what gets rendered is decided down the tree rather than from a list somewhere.

Every UI framework's control hierarchy is the same thing. Panels holding panels holding buttons, and a single `Enabled = false` at the top doing the work.

## What it costs

Composite is lovely right up until leaves and branches genuinely need to behave differently, and then it gets awkward fast.

Ours is already a bit awkward. A leaf needs a URL to navigate to. A branch doesn't, and shouldn't have one. Both have the property, because they're the same class, and the visibility logic ends up checking whether a URL is set as a proxy for "am I a leaf". That works, and it's not what you'd design from scratch.

You can see the strain in that `Visible` property. Before it gets to the recursion there is a chain of separate `if` statements, each checking a different property, working through the combinations of whether there is a URL, whether there is a rule, and whether there are any children. It finishes with a comment reading *"just return true and if it shouldn't be showing figure out what I missed"*. That comment is honest and it's also a signal. Rules that pile up as special cases with a shrug at the bottom want turning into something you can state in one line, and I haven't got round to it.

## In short

If you're drawing a tree of anything, make branches and leaves the same type and let the tree answer questions about itself. Declare the rules where the actual decisions live, and the structure above them takes care of itself.

This is the plain-English version. There's more to it, and the moment your branches and leaves want properly different behaviour rather than just different contents, it's worth asking whether one type is still doing you a favour. But this should cover the basics.
