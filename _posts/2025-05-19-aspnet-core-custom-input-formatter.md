# 🔄 Handling Different Payload Types in ASP.NET Core with a Custom Input Formatter

In modern APIs, especially integrations with third-party platforms, it's common to receive JSON payloads where a field like `"command"` determines the structure of the `"record"` or `"data"` part of the payload.

In this post, I'll show how I **built a custom input formatter** in ASP.NET Core that reads a discriminator field (like `"command"`) and deserializes the request into different .NET types accordingly.

---

## 📦 Example Scenario

You receive POST requests with this structure:

```json
{
  "command": "openObject",
  "record": {
    "data": {
      "attributes": {
        "name": "2",
        "original_identifier": "45678"
      }
    }
  }
}
```

The shape of `"attributes"` depends on the `"command"`. You want to deserialize into different C# classes like:

```csharp
public class OpenObjectPayload
{
    public string Name { get; set; }
    public string OriginalIdentifier { get; set; }
}

public class CloseObjectPayload
{
    public string DoorName { get; set; }
    public DateTime ClosedAt { get; set; }
}
```

---

## ✅ Solution: Custom Input Formatter

ASP.NET Core supports [custom input formatters](https://docs.microsoft.com/en-us/aspnet/core/web-api/advanced/custom-formatters). I'll used this to inspect the incoming JSON and route it to the correct class.

### 1. Create a Base Class

```csharp
public abstract class WebhookRequest
{
    public string Command { get; set; }
}
```

### 2. Define Derived Payload Classes

```csharp
public class OpenObjectRequest : WebhookRequest
{
    public OpenObjectPayload Record { get; set; }
}

public class CloseObjectRequest : WebhookRequest
{
    public CloseObjectPayload Record { get; set; }
}
```

### 3. Create the Custom Input Formatter

```csharp
public class CommandBasedInputFormatter : TextInputFormatter
{
    private readonly Dictionary<string, Type> _typeMappings;

    public CommandBasedInputFormatter()
    {
        SupportedMediaTypes.Add(MediaTypeHeaderValue.Parse("application/json"));
        SupportedEncodings.Add(Encoding.UTF8);

        _typeMappings = new Dictionary<string, Type>(StringComparer.OrdinalIgnoreCase)
        {
            { "openObject", typeof(OpenObjectRequest) },
            { "closeObject", typeof(CloseObjectRequest) }
        };
    }

    protected override bool CanReadType(Type type) => type == typeof(WebhookRequest);

    public override async Task<InputFormatterResult> ReadRequestBodyAsync(InputFormatterContext context, Encoding encoding)
    {
        var request = context.HttpContext.Request;
        using var reader = new StreamReader(request.Body, encoding);
        var json = await reader.ReadToEndAsync();

        var command = JsonDocument.Parse(json).RootElement.GetProperty("command").GetString();

        if (!_typeMappings.TryGetValue(command, out var targetType))
        {
            context.ModelState.AddModelError("command", $"Unknown command: {command}");
            return await InputFormatterResult.FailureAsync();
        }

        var result = JsonSerializer.Deserialize(json, targetType, new JsonSerializerOptions
        {
            PropertyNameCaseInsensitive = true
        });

        return await InputFormatterResult.SuccessAsync(result);
    }
}
```

---

## 🔧 Register the Formatter

In `Startup.cs` (for .NET Core 3.1) or `Program.cs` (for .NET 6+):

```csharp
services.AddControllers(options =>
{
    options.InputFormatters.Insert(0, new CommandBasedInputFormatter());
});
```

---

## 🧪 Use in Controller

```csharp
[HttpPost("webhook")]
public IActionResult HandleWebhook([FromBody] WebhookRequest request)
{
    switch (request)
    {
        case OpenObjectRequest openObject:
            // Handle open object
            break;
        case CloseObjectRequest closeObject:
            // Handle close object
            break;
        default:
            return BadRequest("Unknown command");
    }

    return Ok();
}
```

---

## 🚀 Benefits

✅ Centralised logic for routing based on discriminator fields  
✅ Strongly typed models per command  
✅ Compatible with `application/json` and standard model binding  

---

## 🔚 Conclusion

Using a custom input formatter in ASP.NET Core is a clean and powerful way to handle polymorphic request bodies driven by a `command` or `type` field. This approach keeps the controller actions strongly typed and easier to maintain.
