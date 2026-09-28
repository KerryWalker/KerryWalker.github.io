---
layout: post
title: "One Endpoint for Everything: The Adapter Pattern"
pillar: patterns
series: design-patterns
excerpt: We have an API endpoint that takes a command and answers with a redirect, a notice or a warning. A page just wants the link to an order. The Adapter pattern is the small class in between, and the reason it exists isn't that anybody got the API wrong.
tags:
  - design-patterns
  - csharp
  - dotnet
---

We integrate with a partner system, and one of the things users want is simple enough to describe: they're looking at an order in our software, and they'd like to open that same order in theirs. A link, basically.

Easy request. The complication is that the endpoint which can answer it doesn't deal in links. It deals in commands.

## One endpoint for everything

The API has a single endpoint, and it takes a command:

```csharp
public interface ICommandApi
{
    [Post("/api/v1/command")]
    Task<IApiResponse<CommandResult>> SendCommand(CommandPayload payload, CancellationToken cancelToken);
}
```

That's the whole thing. One method. You fill in a `CommandPayload` saying which command you want and what you want it applied to, post it, and get back a `CommandResult` that is either a redirect, a notice, a warning, or nothing at all.

That shape isn't an accident, and it wasn't my idea. The partner has the same thing, and it's how they let other people extend their software. You register a command, they surface it to a user, and when the user triggers it they call you and you answer: go to this URL, or show this message, or that didn't work. One endpoint and a command name means they can let integrators plug things in without shipping a new endpoint every time somebody does.

I liked it enough to build something similar on our side. Most of what goes through ours is navigation out to them.

So this isn't a story about somebody else's awkward API. We chose this design deliberately and I'd choose it again. It's just that a good shape for an extension point is a bad shape for a caller.

## What a page actually wants

What the calling code wants to say is this:

```csharp
public interface IPartnerNavigationService
{
    Task<CommandResult> GetOrderUrlAsync(int orderId, int siteId, CancellationToken cancelToken);
    Task<CommandResult> GetProductUrlAsync(int productId, int siteId, CancellationToken cancelToken);
    Task<CommandResult> GetContractUrlAsync(int contractId, int siteId, CancellationToken cancelToken);
    Task<CommandResult> GetCustomerUrlAsync(int customerId, int siteId, CancellationToken cancelToken);
}
```

Four methods that each say what they do. No envelope, no HTTP, no commands. A page that needs a link to an order calls the one about orders.

Two interfaces describing the same capability, and they look nothing alike. That gap is the whole job.

## The class in the middle

```csharp
public class PartnerNavigationService : IPartnerNavigationService
{
    public async Task<CommandResult> GetOrderUrlAsync(int orderId, int siteId, CancellationToken cancelToken)
        => await SendNavigationCommandAsync(CommandType.ViewOrderInPartner, orderId, siteId, cancelToken);

    public async Task<CommandResult> GetProductUrlAsync(int productId, int siteId, CancellationToken cancelToken)
        => await SendNavigationCommandAsync(CommandType.ViewProductInPartner, productId, siteId, cancelToken);

    // ...and the other two

    private async Task<CommandResult> SendNavigationCommandAsync(
        CommandType commandType, int entityId, int siteId, CancellationToken cancelToken)
    {
        var subscriptionId = await _subscriptionProvider.GetSubscriptionId();

        var payload = new CommandPayload
        {
            SubscriptionId = subscriptionId,
            Command = commandType,
            ContextId = entityId,
            SiteId = siteId
        };

        var result = await _commandApi.SendCommand(payload, cancelToken);

        if (!result.IsSuccessStatusCode)
        {
            var errorMessage = await result.Error?.GetContentAsAsync<string>() ?? "Unknown error";
            _logger.LogWarning("Command {CommandType} failed with status {StatusCode}: {Error}",
                commandType, result.StatusCode, errorMessage);
            return CommandResult.Warning($"Failed to resolve URL: {errorMessage}");
        }

        return result.Content;
    }
}
```

Ninety lines all in. The four public methods are one-liners that each pick a different command type, and everything else happens once in the private helper.

That's the **Adapter** pattern. One class that implements the interface your code wants, holds the thing you're actually talking to, and translates between them.

## Four things it's quietly doing

It's worth being specific about what the translation involves, because "it wraps the API" undersells it.

**It changes the vocabulary.** "Give me the URL for this order" goes in, "post a command of type `ViewOrderInPartner`" comes out. The calling code never learns that commands exist.

**It supplies context the caller doesn't have.** Every request needs a subscription id. The adapter fetches it. Nothing that wants a link has to know about subscriptions, or hold one, or remember to pass it.

**It strips the envelope.** The API returns `IApiResponse<CommandResult>`. Ours returns `CommandResult`. Status codes, error content and the rest of the HTTP machinery stop at this class and go no further into the application.

**It turns failures into something the app understands.** A non-success response becomes `CommandResult.Warning("Failed to resolve URL: ...")`, logged on the way past. The page that wanted a link gets a result it can render. It doesn't get an exception, and it doesn't get a 502 to interpret.

## Why not just call the endpoint directly?

Because you wouldn't do it once, you'd do it everywhere.

Every screen wanting a link would build its own `CommandPayload`, fetch its own subscription id, pick the right command type, check the status code and decide what to do about a failure. Four or five lines each, repeated across a dozen pages, each one a chance to get the command type wrong or forget the error path.

Then the payload changes. A new required field, a different error shape, a version bump. With an adapter that's one file. Without one it's a search across the solution and a nagging feeling you've missed a caller.

The adapter is also the thing that makes testing bearable. `IPartnerNavigationService` is four methods with no HTTP in sight, so anything that depends on it can be tested with a stub in a couple of lines.

## Where you've already met this

You'll have written one of these without calling it anything:

- Any repository or service class sitting over a third-party SDK, so the rest of the app talks about customers rather than about whatever the vendor decided to call them.
- Every `ILogger` implementation. Serilog, NLog and the rest have their own APIs, and an adapter makes them all look like the one interface your code uses.
- `System.IO.Stream` over a file, a network socket or a block of memory. Three completely different things behind one set of methods.

The pattern is the same each time: something exists, it's useful, and it doesn't talk the way you'd like. So you write the small class that does.

## What it costs

An extra layer, and two temptations that come with it.

The first is that an adapter stops adapting and turns into a pass-through. If every method just forwards a call unchanged and hands back the same type, you've added a file and a hop and gained nothing. Ours earns its place because the two interfaces genuinely differ. A thin wrapper over an API that already suits you is ceremony.

The second is that you stop halfway, which is what we did. Look at the return type on that clean interface: it's still `CommandResult`. The caller no longer has to know about HTTP, but it does still have to know that a URL arrives as a `Message` on a result whose `ResultType` says `Redirect`. The adapter stripped one envelope and kept the other. Nobody has minded enough to fix it, and that is usually how these things stay.

The last cost is ordinary maintenance. When a new command gets added, somebody has to add a method here. That's a fair trade for having one place to add it, but it isn't free.

## In short

When something does what you need but says it in the wrong shape, don't teach your whole application to speak that shape. Write one class that does, and let everything else carry on speaking its own.

And the wrong shape isn't always somebody else's fault. A generic command envelope is exactly right for an extension point, where the whole idea is that you don't know in advance what will be plugged into it. It's exactly wrong for a page that wants one specific thing. Both are true of the same endpoint at the same time, which is the reason the class in the middle exists.

This is the plain-English version. There's more to it, and the line between an adapter and a facade is blurrier than most explanations admit. But this should cover the basics.

One loose end, and it's a word rather than a design. I've spent this whole post using "command" to mean an envelope going over the wire. There's a pattern with the same name that means something quite different, and it sits behind every button on every menu in the app. That's next.
