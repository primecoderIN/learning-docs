# Chapter 11: ASP.NET Core Fundamentals

## 1. Learning Objectives
By the end of this chapter, you will be able to:
- Understand the architecture of RESTful APIs and HTTP semantics.
- Define HTTP endpoints using ASP.NET Core Minimal APIs.
- Extract data from the HTTP Request (Route parameters, Query Strings, and JSON Bodies).
- Serialize C# objects to JSON Responses automatically.
- Return appropriate HTTP Status Codes using the `IResult` interface.

## 2. Introduction

In previous chapters, we focused on how C# code runs on the CPU and interacts with memory. But an enterprise application does not exist in a vacuum. It must communicate with external systems—mobile apps, single-page web applications (React/Angular), and third-party servers.

The universal language of modern distributed systems is **HTTP (Hypertext Transfer Protocol)**. 
**ASP.NET Core** is the web framework built into .NET that allows you to easily map incoming HTTP requests over the network to specific C# methods in your application.

Before we dive into the extreme performance internals of Kestrel and the Middleware pipeline in the next chapter, we must first master the basics: How do we expose our C# logic to the internet?

## 3. RESTful API Design and HTTP Verbs

Most modern web services follow **REST (Representational State Transfer)** principles. In REST, everything is treated as a "Resource" (e.g., a Charger, a Tenant, an Invoice).

Clients interact with these resources using standard HTTP Methods (Verbs):
- **GET:** Retrieve a resource (Read-only, idempotent).
- **POST:** Create a new resource.
- **PUT:** Completely replace an existing resource.
- **PATCH:** Partially update an existing resource.
- **DELETE:** Remove a resource.

### The Anatomy of an HTTP Request
When a client (like Postman or a web browser) sends a request to your server, it sends raw text that looks like this:

```http
POST /api/chargers HTTP/1.1
Host: api.evplatform.com
Content-Type: application/json
Authorization: Bearer xyz123

{
    "id": "CHG-100",
    "maxKw": 150
}
```

Your job as an ASP.NET Core developer is to configure the framework to listen for this text, extract the JSON, map it to a C# object, run your business logic, and return an HTTP Response.

## 4. Routing and Minimal APIs

Routing is the process of mapping a specific URL (like `/api/chargers`) to a specific block of C# code. 
Since C# 10, the most efficient and readable way to do this is using **Minimal APIs**.

```csharp
using Microsoft.AspNetCore.Builder;

// 1. Initialize the Web Application
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// 2. Define a Route (GET request to the root URL)
app.MapGet("/", () => "Welcome to the EV Platform API!");

// 3. Start listening for incoming HTTP requests
app.Run();
```

If you run this code and navigate to `http://localhost:5000/` in your browser, the ASP.NET Core engine will match the route `/` to the lambda function `() => "Welcome..."` and return the string to the browser.

## 5. Model Binding: Extracting Data

Endpoints usually need data from the client to perform their work. Data can arrive in three primary ways:
1. **Route Parameters:** Embedded directly in the URL path.
2. **Query Strings:** Appended to the end of the URL after a `?`.
3. **Request Body:** Sent as JSON inside a POST or PUT request.

ASP.NET Core uses a feature called **Model Binding** to automatically extract this data and convert it into strongly-typed C# variables.

### Route Parameters
Route parameters are defined using curly braces `{}` in the route template.

```csharp
// URL: GET /api/chargers/CHG-100
app.MapGet("/api/chargers/{id}", (string id) => 
{
    // The framework automatically extracts "CHG-100" and passes it to 'id'
    return $"Fetching details for charger {id}";
});
```

### Query Strings
If a parameter is defined in the C# method signature but is *not* in the route template, ASP.NET Core assumes it comes from the Query String.

```csharp
// URL: GET /api/chargers?region=Europe&isActive=true
app.MapGet("/api/chargers", (string region, bool isActive) => 
{
    // The framework automatically parses the URL and converts 'true' to a boolean!
    return $"Searching chargers in {region}. Active only: {isActive}";
});
```

### JSON Request Bodies
For complex data, clients send JSON in the HTTP Body. You simply define a C# `class` or `record`, and pass it as a parameter. The framework will automatically deserialize the JSON.

```csharp
public record CreateChargerDto(string Id, decimal MaxKw);

// URL: POST /api/chargers
// Body: { "id": "CHG-200", "maxKw": 50 }
app.MapPost("/api/chargers", (CreateChargerDto newCharger) => 
{
    // 'newCharger' is a fully instantiated C# object
    Console.WriteLine($"Saving charger {newCharger.Id} to database...");
    
    return "Charger created successfully.";
});
```

## 6. Returning JSON and Status Codes

Returning plain strings is not enough for an API. You must return structured data (usually JSON) and appropriate HTTP Status Codes (e.g., 200 OK, 404 Not Found, 400 Bad Request).

ASP.NET Core uses the static `Results` class, which implements the `IResult` interface, to format responses correctly.

```csharp
app.MapGet("/api/chargers/{id}", (string id) => 
{
    if (id != "CHG-100")
    {
        // Generates an HTTP 404 Not Found response
        return Results.NotFound(new { Error = "Charger does not exist." });
    }

    // Creating an anonymous C# object
    var chargerData = new { Id = id, Status = "Online", MaxKw = 150 };

    // Generates an HTTP 200 OK response, and automatically serializes the object to JSON
    return Results.Ok(chargerData);
});
```

## 7. Real Production Case Study: EV Configuration API

Let's combine everything into a realistic segment of our EV Platform. We will create an API to manage the configuration of a specific charger. We will use asynchronous programming (`await`), routing, model binding, and proper status codes.

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using System.Threading.Tasks;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// Mock database dictionary
var _database = new System.Collections.Concurrent.ConcurrentDictionary<string, ChargerConfig>();

// GET: Retrieve configuration
app.MapGet("/api/v1/chargers/{id}/config", async (string id) =>
{
    // Simulate async database I/O
    await Task.Delay(50); 

    if (_database.TryGetValue(id, out var config))
    {
        return Results.Ok(config); // HTTP 200 with JSON payload
    }
    
    return Results.NotFound(); // HTTP 404
});

// PUT: Update or Create configuration
app.MapPut("/api/v1/chargers/{id}/config", async (string id, ChargerConfig newConfig) =>
{
    // Validation
    if (newConfig.MaxAmpLimit <= 0)
    {
        return Results.BadRequest(new { Error = "Amps must be greater than zero." }); // HTTP 400
    }

    await Task.Delay(50); // Simulate DB Write
    
    _database[id] = newConfig;

    // HTTP 204 No Content (Standard for successful PUT requests that don't return data)
    return Results.NoContent(); 
});

app.Run();

// Domain Record
public record ChargerConfig(string FirmwareVersion, int MaxAmpLimit);
```

## 8. Common Mistakes

| Level | Mistake | Impact | How to Avoid |
|-------|---------|--------|--------------|
| Beginner | Returning `null` from an endpoint when data isn't found. | Client receives an HTTP 200 OK with an empty body, causing parsing errors. | Explicitly return `Results.NotFound()` when an entity does not exist. |
| Intermediate| Trying to read the Request Body twice. | Exception thrown. The request body is a forward-only stream. | Let Model Binding handle the body extraction into a C# object, and pass that object around. |
| Senior | Putting try/catch blocks in every single endpoint. | Massive code duplication. | Use global Exception Handling Middleware (covered in Chapter 12) to catch errors and return HTTP 500 automatically. |
| Architect | Exposing internal Database Entities (EF Core classes) directly in the API return. | Leaking sensitive schema data (like passwords or internal IDs) to the public internet. | Always map Database Entities to Data Transfer Objects (DTOs) before returning them via `Results.Ok()`. |

## 9. Summary
In this chapter, we bridged the gap between C# logic and the web. We learned how to map URLs to C# delegates using Minimal APIs, how to automatically extract JSON payloads using Model Binding, and how to respond with correct HTTP Status Codes. 

This is the high-level, developer-friendly face of ASP.NET Core. But beneath this elegant syntax is an incredibly complex, high-performance engine. In the next chapter, we will strip away the magic and look at the underlying Architecture and Internals: Kestrel, Middleware pipelines, and the Dependency Injection container.
