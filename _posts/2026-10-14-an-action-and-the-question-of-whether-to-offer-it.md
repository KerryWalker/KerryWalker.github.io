---
layout: post
published: false
title: "An Action, and the Question of Whether to Offer It: The Command Pattern"
pillar: patterns
series: design-patterns
excerpt: Every screen has a menu of things you can do to the record you're looking at. Packaging each one as an object is the Command pattern. The part most explanations skip is that you need a second object to say whether it should be there at all.
tags:
  - design-patterns
  - csharp
  - dotnet
---

[Last time]({% post_url 2026-10-09-one-endpoint-for-everything %}) I used "command" to mean an envelope posted to an endpoint. Here it means something else entirely, and the two aren't related. Same word, two ideas, both of them in our code, which is worth knowing before somebody says "the command handler" and you go looking in the wrong place.

This one is the pattern.

Nearly every screen in the app I work on has a menu hanging off it. You're looking at a customer, and down the side is a list of things you can do to that customer. Open the linked product. View the contract. Change the payment details. Twenty-odd items on some screens.

They have nothing in common. Some open a dialog, some navigate away, some call an external system, some just show a message. What they do share is that the menu has to deal with all of them the same way, and that's the problem worth solving.

## An action as an object

The obvious approach is a method per action and a `switch` on whichever one was clicked. That works until the menu is used on six screens with different items on each, and then it doesn't.

So each action is a class instead:

```csharp
public abstract class ItemBehaviour : IItemBehaviour
{
    public virtual async Task DoAction()
    {
        await this.Status.SetCompleted();
    }

    public CancellationTokenSource CancelTokenSource { get; private set; }
    public ActionStatus Status { get; set; }
}
```

`DoAction` is the whole interface as far as the menu is concerned. It doesn't know or care what happens inside.

A real one looks like this:

```csharp
public class ViewInPartnerBehaviour : ItemBehaviour
{
    public ViewInPartnerBehaviour(
        ICustomerViewModel viewModel,
        IMessageBoxService messageBoxService,
        ILogger<ViewInPartnerBehaviour> logger,
        IPartnerNavigationService navigationService,
        INavManager navManager) : base(logger, null, messageBoxService, null)
    {
        // ...
    }

    public override async Task DoAction()
    {
        var productId = _viewModel.Item.ProductId ?? 0;
        if (productId == 0)
        {
            await MessageBoxService.ShowAsync("View Product",
                "No product is associated with this account.",
                MessageBoxButtons.Ok, MessageBoxIcon.Warning);
            return;
        }

        // ask the partner for a URL and navigate to it
    }
}
```

That's the **Command** pattern: an action packaged as an object, with its dependencies injected, exposing one method that runs it. The menu holds a list of them and calls `DoAction()` on whichever one was clicked.

It's also using [the adapter]({% post_url 2026-10-09-one-endpoint-for-everything %}) to get the URL, which is quite a neat illustration of how these things stack up in real code. A command that calls an adapter that wraps an endpoint.

## The half that usually gets left out

Every explanation of Command I've read stops there. An object, a method, run it. But a menu needs to know things about an action *before* anyone clicks it.

Should this item appear at all? This customer has no linked product, so "View Product" is meaningless. Should it be greyed out? The form has unsaved changes, so navigating away would lose them.

You cannot answer either of those from inside a class whose only method does the thing. So there's a second object:

```csharp
public class ViewInPartnerRules : PartnerActionRules
{
    protected override int? GetSiteId() => _viewModel.Item?.SiteId;
    protected override bool AdditionalVisibilityConditions => _viewModel.Item?.ProductId > 0;
    public override bool Enabled => !_viewModel.FormModified;
}
```

Three lines of actual decision. Show it if there's a product linked. Enable it if the form isn't dirty.

And a base that says what every rule has to answer:

```csharp
public abstract class ActionRules : IActionRules
{
    public abstract bool Visible { get; }
    public abstract bool Enabled { get; }
    public virtual FeatureFlag FeatureFlag => FeatureFlag.CreateEmpty();

    // Most rules answer from the view model alone and have nothing to fetch.
    public virtual Task PrepareAsync() => Task.CompletedTask;
}
```

Every action is a pair. One object knows how to do it, the other knows whether you should be allowed to.

## Why not just put the properties on the command?

You could. `Visible` and `Enabled` as properties on `ItemBehaviour` and no second class.

The reason we don't is that they get built at different times, for different reasons. The rules are evaluated constantly, every time the screen redraws or the underlying record changes, for every item in the menu. The behaviour is constructed once and used if somebody clicks it. Putting them together means dragging a command's worth of dependencies into existence just to work out whether to grey out a menu item.

Look at the constructors above and it's obvious. The behaviour takes a message box service, a logger, a navigation service and a nav manager, because doing the thing needs all of them. The rules take a view model. Asking the question is cheap, and keeping it cheap matters when you're asking it twenty times a second.

It also keeps each class honest. A rules class that can't do anything can't quietly start doing something.

## What two hundred of them teach you

I didn't design this. It was there before me. What I have done is write a lot of them, and the tests for most of what I wrote. The app is up to a couple of hundred behaviours now, with getting on for twice as many rules.

Two things you notice at that volume.

**The pattern is the documentation.** Adding a new menu item means two files in a folder named after the action, matching everything around them. There's nothing to decide and nothing to look up. I've onboarded people onto this by pointing at a folder and saying "like that one", and it works, which is not true of much code.

**Consistency is worth more than elegance.** Some of these commands are three lines and having two classes for them is plainly over the top. It's still better than having a tasteful judgement call at every menu item about whether this one deserves the full treatment. A hundred boring identical things beat eighty tidy ones and twenty special cases, because nobody has to hold the exceptions in their head.

## Where you've already met this

`ICommand` in WPF, which is the same idea with the rules folded in as `CanExecute`. Worth comparing: one interface, `Execute` plus `CanExecute`, and for most UI work that's the right call. Ours split them because our `CanExecute` needed to be much cheaper than our `Execute`.

Another codebase I work in has both sides of that argument sitting in the same folder. One class takes the action and a predicate:

```csharp
public DelegateCommand(Action<T> execute, Predicate<T> canExecute)
```

The other takes only the action, and answers the question the same way every time:

```csharp
public bool CanExecute(object parameter)
{
    return true;
}
```

There's nothing wrong with the second one. Plenty of buttons are always available. But it still has to declare `CanExecuteChanged`, because the interface insists, and nothing in it ever raises that event. The member survived and the reason for it didn't. Drop the question and what's left is the ceremony of having been asked.

Undo stacks are Command's other big use. If an action is an object, you can keep the object, and if it knows how to reverse itself you get undo nearly for free.

Any background job queue too. A job is an action you've packaged up and put somewhere so something else can run it later, which is exactly what a command is for.

## What it costs

Two files per menu item, and a folder per action. For an app with several hundred menu items that's a lot of small classes, and the first time you go looking for where something happens you will not enjoy it.

The other cost is that the pair can drift. Nothing enforces that a behaviour and its rules agree about the world. It's possible to have rules saying an item is visible and a behaviour that immediately puts up a message box explaining it can't do anything, which, as it happens, is exactly what the example above does when there's no product linked. Belt and braces, or a duplicated condition, depending on how charitable you're feeling.

## In short

Wrap an action in an object when something else needs to hold it, list it, delay it or reverse it. Then ask whether that something also needs to reason about the action without running it, because if it does, that's a second object and not a property.

This is the plain-English version. There's more to it, and the moment you want undo you'll need commands that know how to reverse themselves, which is a different conversation. But this should cover the basics.
