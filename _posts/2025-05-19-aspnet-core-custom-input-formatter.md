---
layout: post
title: "One Endpoint, Many Payload Shapes: Custom Input Formatters in ASP.NET Core"
excerpt: A partner posts everything to one endpoint, and a field buried in the payload decides what the rest of it means. Model binding can't help you, so you write the bit that can.
tags:
  - csharp
  - dotnet
  - aspnetcore
---

We take webhooks from a partner system. Everything arrives at one endpoint, and the body looks roughly like this:

```json
{
  "command": "openObject",
  "record": {
    "data": {
      "type": "reservation",
      "attributes": {
        "name": "Site 14",
        "original_identifier": "45678"
      }
    }
  }
}
```

The useful part is in `attributes`, and its shape depends entirely on `type`. A reservation has dates and a pitch. A customer has an email address and a billing reference. They share nothing beyond the envelope they arrive in.

So one endpoint, one content type, and a dozen different objects coming through it. The question is how you get the right .NET type out the other end without a `switch` in every controller action.

## Why model binding can't do it on its own

Model binding decides what to deserialise into before it looks at the body. The method signature says `[FromBody] SomethingRequest`, and that is the end of the conversation. It cannot read a field, change its mind, and bind to a different class.

You can bind to `JsonDocument` and sort it out yourself in the action, and plenty of code does. It works, and it puts deserialisation logic in every action that needs it, which is the thing worth avoiding.

What you want is for the decision to happen once, before the controller is involved, so the action can take a typed object and get on with it. That is what an input formatter is for.

## The formatter

An input formatter is the thing MVC uses to turn a request body into an object. There is already one handling `application/json`, and you can put your own in front of it for the types you care about:

```csharp
public class PayloadInputFormatter : TextInputFormatter
{
    public PayloadInputFormatter(ILogger<PayloadInputFormatter> logger)
    {
        _logger = logger;

        SupportedMediaTypes.Add("application/json");
        SupportedEncodings.Add(Encoding.UTF8);
        SupportedEncodings.Add(Encoding.Unicode);
    }

    protected override bool CanReadType(Type type) =>
        type == typeof(CommandPayload) || type == typeof(WebHookPayload);
}
```

`CanReadType` is the important line. It is how you claim only the types you handle and leave everything else alone, which matters because the moment you register a formatter it is offered every request body in the application.

## Reading it more than once

Here is the part the whole post is about.

You cannot deserialise into the right type until you know what the right type is, and you cannot know that until you have read part of the body. So you read it more than once.

```csharp
public override async Task<InputFormatterResult> ReadRequestBodyAsync(
    InputFormatterContext context, Encoding encoding)
{
    using var reader = new StreamReader(context.HttpContext.Request.Body, encoding);
    var body = await reader.ReadToEndAsync();

    using var root = JsonDocument.Parse(body);

    // First pass: the envelope, into the type the action asked for
    var request = (Payload)JsonSerializer.Deserialize(body, context.ModelType, jsonOptions);

    // The interesting part is nested, so pull just that out
    var dataJson = root.RootElement
        .GetProperty("record")
        .GetProperty("data")
        .GetRawText();

    // Second pass: deserialise it as the base type, purely to read the discriminator
    var data = (IData)JsonSerializer.Deserialize(dataJson, typeof(Data), jsonOptions);

    if (!ObjectTypeMap.TryGetValue(data.Type, out var targetType))
    {
        _logger.LogWarning("Unknown payload type '{Type}' received.", data.Type);
        return await InputFormatterResult.FailureAsync();
    }

    // Third pass: now deserialise the same JSON into the type we resolved
    request.Record.ParsedData = (IData)JsonSerializer.Deserialize(dataJson, targetType, jsonOptions);

    return await InputFormatterResult.SuccessAsync(request);
}
```

That is three passes over overlapping JSON, which sounds like it ought to be a problem. It isn't, because no two of them are doing the same job.

The first builds the envelope, which is the same for everything. The second reads one field, and reading it through the serialiser rather than poking at `JsonDocument` means the naming policy and converters apply consistently. The third is the only one that needs the resolved type.

`GetRawText()` is what makes it cheap enough not to care. It hands back the JSON for that node as a string, so the second and third passes work on the nested fragment rather than the whole body.

The map itself is unremarkable, and that is the point:

```csharp
private static readonly Dictionary<string, Type> ObjectTypeMap = new()
{
    { "reservation", typeof(ReservationData) },
    { "customer",    typeof(CustomerData) },
    // ...
};
```

Adding a payload type is one entry and one class.

## The options object, and why it is static

```csharp
private static readonly JsonSerializerOptions jsonOptions = new()
{
    PropertyNameCaseInsensitive = true,
    PropertyNamingPolicy = JsonNamingPolicy.SnakeCaseLower,
    ReferenceHandler = ReferenceHandler.IgnoreCycles,
    Converters = { new JsonStringEnumConverter() },
};
```

Two things here are worth more than they look.

`PropertyNamingPolicy = JsonNamingPolicy.SnakeCaseLower` is what maps `original_identifier` to `OriginalIdentifier`. It is easy to assume `PropertyNameCaseInsensitive` already covers that, and it does not. Case insensitivity matches `name` to `Name`. It will not match an underscore to a capital letter, because an underscore is not a case. Get this wrong and every snake_case property binds as null while the camelCase ones work, which is a confusing half hour.

And it is `static`. A `JsonSerializerOptions` is meant to be built once and shared, partly because it caches serialisation metadata and partly because mutating one after it has been used is not safe from several requests at once. Build it in a static field, populate the converters there, and never touch it again.

## Registering it

```csharp
services.AddControllers(options =>
{
    options.InputFormatters.Insert(0, new PayloadInputFormatter(logger));
});
```

`Insert(0, ...)` rather than `Add`, because formatters are tried in order and the built-in JSON one will happily claim the request first.

The controller then looks like nothing in particular, which is the whole return on the exercise:

```csharp
[HttpPost("webhook")]
public async Task<IActionResult> HandleWebhook([FromBody] WebHookPayload payload)
{
    var handler = _handlerFactory.Resolve(payload.Record.ParsedData);
    return Ok(await handler.Process(payload));
}
```

No reading the body, no `JsonDocument`, no switch on a string. The action gets a typed object and dispatches it.

## When you should not do this

Since .NET 7, `System.Text.Json` handles polymorphic deserialisation on its own:

```csharp
[JsonPolymorphic(TypeDiscriminatorPropertyName = "type")]
[JsonDerivedType(typeof(ReservationData), "reservation")]
[JsonDerivedType(typeof(CustomerData), "customer")]
public abstract class Data { }
```

If that works for you, use it. It is less code, there is no formatter to register, and the mapping sits next to the types it describes.

It did not work for us for one reason: the discriminator is not where `[JsonPolymorphic]` expects it. It wants the type field at the root of the object being deserialised. Ours is two levels down, inside an envelope we do not control, and the attribute has no way to reach it.

That is the test. If you own the payload shape, or the discriminator is at the root of the thing it describes, use the attributes. Write a formatter when the payload is somebody else's and the decision you need to make depends on something buried in it.

## In short

An input formatter is the right place to decide what type a request body becomes, because it runs before model binding has committed to an answer and it runs once rather than in every action.

The technique is just reading the body twice: once to find out what you are looking at, once to turn it into that. Everything else is a dictionary.
