---
layout: post
title: "What Happens on the Way Back Out: Middleware and the Decorator Pattern"
pillar: patterns
series: design-patterns
excerpt: "The third thing that gets called a chain, and the one most .NET developers picture first. Every link sees the request go in and the response come out, and that second half is what makes it a different pattern with a different name."
tags:
  - design-patterns
  - csharp
  - dotnet
---

Two earlier posts in this series pulled apart things that get called chains. [The first]({% post_url 2026-11-02-the-chain-that-knows-when-to-stop %}) was a line of searchers where the first one to find something ends it. [The second]({% post_url 2026-11-07-steps-not-alternatives %}) was a line of import stages where every one takes a turn and the rows move forward. The wiring looks the same in both.

There is a third shape that gets called a chain more than either of them, and it is the one most .NET developers picture when they hear the word. Request handlers on an `HttpClient`. Middleware in a web application. A line of things, each holding the next, each passing the call along.

It is not either of the first two, and the reason is a question neither of them ever had to answer: what happens on the way back?

## The problem it was solving

The app I work on talks to a partner's booking API. Every request to it needs a bearer token, and the token belongs to a tenant rather than to us, so there is one per customer account and each one expires on its own schedule. A refresh token can fetch a new one, and that can expire too.

There are a dozen or so separate clients talking to that API, one per area of it, and every one of them needs the same thing done to every request. Add the right accept headers. Work out which tenant this call is for. Find that tenant's token, refresh it if it has expired, put it on the request.

A dozen clients, every method on each of them, and the same twenty lines. You do not want that in the clients. You want it somewhere the clients cannot see.

## A handler on the way in

.NET gives you a place for this. `HttpClient` sends through a line of `DelegatingHandler` objects, each holding the next, and each one gets the request before the real network call happens. Here is ours, trimmed:

```csharp
public class PartnerAuthHandler : DelegatingHandler
{
    private readonly IAuthenticationService _authentication;
    private readonly ITokenService _tokens;

    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken cancellationToken)
    {
        request.Headers.Add("accept", "application/vnd.api+json");

        string tenantId = null;
        if (request.Options.TryGetValue(RequestKeys.TenantId, out tenantId))
        {
            await SetToken(tenantId, request, cancellationToken);
        }

        var response = await base.SendAsync(request, cancellationToken);

        // ...

        return response;
    }

    private async Task SetToken(string tenantId, HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        if (!await _authentication.IsAccessTokenValid(tenantId)
            && _authentication.HasValidRefreshToken(tenantId))
        {
            await _authentication.RefreshAccessToken(tenantId, cancellationToken);
        }
        var token = await _tokens.GetAccessToken(tenantId);
        request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);
    }
}
```

So far this looks like a pipeline stage. Something arrives, the handler does its bit to it, and `base.SendAsync` passes it to whatever is next in the line. The clients call the API as if none of this existed, and the token appears on the request before it leaves the process.

If that were all it did, the previous post would cover it. It is not all it does.

## And on the way back out

Here is the part I cut:

```csharp
var response = await base.SendAsync(request, cancellationToken);

if (response.StatusCode == HttpStatusCode.Unauthorized
    || response.StatusCode == HttpStatusCode.Forbidden)
{
    if (!string.IsNullOrEmpty(tenantId))
    {
        response.Dispose();
        await SetToken(tenantId, request, cancellationToken);
        return await base.SendAsync(request, cancellationToken);
    }
}
return response;
```

The call to `base.SendAsync` is not the end of the handler. It is the middle. Everything above it runs on the way in, and everything below it runs on the way back out, with the response in hand.

That changes what the handler can do. It can look at the answer. If the partner says the token was no good after all, which happens when one has been revoked at their end without ours knowing, the handler throws the response away, fetches a fresh token, and sends the request again. The caller asked once and gets one response. It never finds out that two calls were made.

Neither of the previous two shapes could do this. In the search chain, a link handed the request on and that was the end of its involvement. In the import pipeline, a stage pushed rows to the next queue and never saw them again. Here, control comes back. The handler is wrapped around the rest of the line, not sat in front of it.

## That is a Decorator

The name for an object that wraps another object of the same type, and adds behaviour before and after handing off to it, is the **Decorator** pattern. The thing being wrapped does not know it is wrapped. The thing calling in does not know either. From both sides it looks like the plain object.

`DelegatingHandler` is that pattern with the framework's name on it. The component is `HttpMessageHandler`. The real one at the bottom does the network call. Every delegating handler holds one of them as its inner handler, overrides the same `SendAsync`, and decides what to do before and after calling through.

Middleware is the same thing again, which is why people who have never heard of the pattern already write it:

```csharp
app.Use(async (context, next) =>
{
    // on the way in
    await next();
    // on the way back out
});
```

The `await next()` in the middle is the hinge, and it is the only line that matters for telling the shapes apart. Code before it runs as the request goes in. Code after it runs as the response comes out. Leave the second half empty and you have built a pipeline stage. Use it and you have built a decorator.

## Wrapping a wrapper

The reason this pattern gets its own name rather than being filed under "a function that calls another function" is that decorators stack, and the stacking order means something.

Ours is registered like this, once per client:

```csharp
services.AddRefitClient<IPartnerBookingsApi>(settings)
    .AddHttpMessageHandler<PartnerAuthHandler>()
    .AddPolicyHandler(PartnerRetryPolicies.GetCombinedPolicy(...));
```

The second line is Polly. We wrote the policy, which retries on a 429 or a transient failure with a back-off, and `AddPolicyHandler` wraps that policy in a delegating handler of Polly's own and registers it. So what sits in the line is two handlers of the same kind: one we wrote, and one the library wrapped round our policy. The order they are registered in is the order they nest, and the first registered is outermost. So the retry handler sits inside the auth handler, wrapped around the real network call.

Which means there are two retry loops on every request, and they never fight. A rate limit or a server error comes back from the network, the policy catches it on its way out, waits, and sends again with the same token. The auth handler never sees any of that. A 401 is not something the policy handles, so it passes straight through, reaches the auth handler on its way out, and that one refreshes the token and resends. That resend goes back in through the policy, so it gets the rate limit protection as well.

Each wrapper handles one concern and sees only what the wrappers inside it let out. That is the thing the pattern buys you, and it is also the thing that goes wrong when the order is not what you thought. Swap those two lines and the retry policy would be outside the auth handler, retrying a request whose token the auth handler had already given up on. Nothing would fail to compile. It would just behave differently, and the order is not written anywhere except in that registration.

## Where you have already met this

Streams. `GZipStream` wraps a `FileStream`, and a `BufferedStream` can wrap either, and all three are a `Stream`. Code that reads from one cannot tell how many layers are underneath, and each layer does its work on the bytes going in and coming out. That is the pattern in the base class library, and it predates most of the people using it.

And the one above: ASP.NET Core middleware, where the whole request pipeline is decorators all the way down, and the order of `app.Use` calls is the order of nesting. Authentication before authorisation, exception handling outside everything, and the reason the docs are so insistent about the order is exactly the reason above.

## What it costs

The order is invisible from the handlers themselves. Reading `PartnerAuthHandler` tells you nothing about what it is inside or outside of. You have to find the registration, and in a large project that is a file nobody opens on purpose.

Retrying on the way out is only safe when you know the inner call did nothing. A 401 is safe, because the partner rejected the request before acting on it. The same trick on a timeout would be a gamble, because the request might have gone through and the response got lost, and now a booking exists twice. The pattern makes resending easy. It does not make it correct.

Sending the same request twice also relies on the body being re-readable. Ours are small JSON payloads held in memory, so it works. A streamed body would have been consumed by the first attempt and the second would go out empty.

And when it goes wrong, the stack trace is `SendAsync` calling `SendAsync` calling `SendAsync`, and working out which layer did what means reading them all.

## In short

Three shapes that all get called chains, and the wiring will not tell you which you have.

If a link can answer and end it, it is Chain of Responsibility. If every link takes a turn and the data moves forward, it is a pipeline. If every link sees the request go in and the response come back out, and can act on both, it is a Decorator, and middleware is the name it goes by when a framework hands it to you.

The giveaway is whether there is a second half. Code after the call through is what makes a decorator, and stacking them is what makes the order matter. Keep each layer to one concern, be deliberate about which one is outermost, and only retry from a layer that knows the layer inside it did nothing.
