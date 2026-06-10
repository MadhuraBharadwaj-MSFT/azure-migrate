# C# — Azure Functions Isolated Worker Model Triggers & Bindings

## Cloud Run Functions Migration Rules

> Full scenario guidance → [cloudrun-functions-to-functions.md](../cloudrun-functions-to-functions.md)

C#-specific Cloud Run functions rules:
- **Drop `Google.Cloud.Functions.Hosting`** and `Google.Cloud.Functions.Framework` NuGet packages from `.csproj`
- **Replace `IHttpFunction` / `ICloudEventFunction<T>` interfaces** with Azure Functions Isolated Worker method attributes (`[Function]` + trigger attribute)
- **Use Isolated Worker model** (`Microsoft.Azure.Functions.Worker`) — do NOT use the in-process model for new code
- **HTTP signature change**:
  - GCP: `Task HandleAsync(HttpContext context)` (writes to `context.Response`)
  - Azure: `Task<HttpResponseData> RunAsync([HttpTrigger] HttpRequestData req, FunctionContext context)` (returns response)
- **CloudEvent unwrap**:
  - GCP: `Task HandleAsync(CloudEvent cloudEvent, MessagePublishedData data, ...)` (Google generates strongly-typed `data` param per event type)
  - Azure: trigger binding delivers unwrapped payload — `[ServiceBusTrigger] string message`, `[BlobTrigger] byte[] blob`, etc.
- **Drop the `Functions Framework` startup**: `await new HostBuilder().UseFunctionsFramework().Build().RunAsync()` → replace with the Azure Functions isolated worker `Program.cs` (`HostBuilder().ConfigureFunctionsWebApplication().Build().Run()`)

### Canonical Side-by-Side — HTTP Function

```csharp
// ❌ BEFORE — Cloud Run functions (C#)
using Google.Cloud.Functions.Framework;
using Microsoft.AspNetCore.Http;

public class HelloHttp : IHttpFunction {
  public async Task HandleAsync(HttpContext context) {
    var name = context.Request.Query["name"].FirstOrDefault() ?? "World";
    await context.Response.WriteAsync($"Hello, {name}!");
  }
}

// ✅ AFTER — Azure Functions Isolated Worker (C#)
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;

public class HelloHttp {
  [Function("helloHttp")]
  public async Task<HttpResponseData> RunAsync(
      [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post")] HttpRequestData req,
      FunctionContext context) {
    var name = System.Web.HttpUtility.ParseQueryString(req.Url.Query)["name"] ?? "World";
    var res = req.CreateResponse(System.Net.HttpStatusCode.OK);
    await res.WriteStringAsync($"Hello, {name}!");
    return res;
  }
}
```

### Canonical Side-by-Side — Pub/Sub CloudEvent → Service Bus Queue

```csharp
// ❌ BEFORE — Cloud Run functions consuming Pub/Sub
using CloudNative.CloudEvents;
using Google.Cloud.Functions.Framework;
using Google.Events.Protobuf.Cloud.PubSub.V1;

public class HandleMessage : ICloudEventFunction<MessagePublishedData> {
  public Task HandleAsync(CloudEvent cloudEvent, MessagePublishedData data, CancellationToken ct) {
    var payload = data.Message.TextData;   // already-decoded by the strongly-typed binding
    Console.WriteLine($"Got message {data.Message.MessageId}: {payload}");
    return Task.CompletedTask;
  }
}

// ✅ AFTER — Azure Functions Isolated Worker Service Bus trigger
public class HandleMessage {
  [Function("handleMessage")]
  public void Run(
      [ServiceBusTrigger("messages", Connection = "ServiceBusConnection")] string message,
      FunctionContext context) {
    var logger = context.GetLogger<HandleMessage>();
    logger.LogInformation("Got message: {Message}", message);
  }
}
```


> **Model**: .NET isolated worker model (recommended). Uses attributes on methods/parameters.
> Import: `using Microsoft.Azure.Functions.Worker;`

## HTTP Trigger

```csharp
[Function("HttpFunction")]
public static HttpResponseData Run(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post")] HttpRequestData req,
    FunctionContext context)
{
    var response = req.CreateResponse(HttpStatusCode.OK);
    response.WriteString("Hello!");
    return response;
}
```

## Blob Storage

```csharp
// Trigger (EventGrid source)
[Function("BlobTrigger")]
public static void Run(
    [BlobTrigger("samples-workitems/{name}", Source = BlobTriggerSource.EventGrid,
     Connection = "AzureWebJobsStorage")] string blobContent,
    string name, FunctionContext context)
{
    context.GetLogger("BlobTrigger").LogInformation($"Blob: {name}");
}

// Input
[BlobInput("input/{name}", Connection = "AzureWebJobsStorage")] string inputBlob

// Output
[BlobOutput("output/{name}", Connection = "AzureWebJobsStorage")] out string outputBlob
```

## Queue Storage

```csharp
// Trigger
[Function("QueueTrigger")]
public static void Run(
    [QueueTrigger("myqueue-items", Connection = "AzureWebJobsStorage")] string message,
    FunctionContext context)
{
    context.GetLogger("Queue").LogInformation($"Message: {message}");
}

// Output (via return type)
[Function("QueueOutput")]
[QueueOutput("outqueue", Connection = "AzureWebJobsStorage")]
public static string Run(
    [HttpTrigger(AuthorizationLevel.Anonymous, "post")] HttpRequestData req)
{
    return "queue message";
}
```

## Timer

```csharp
[Function("TimerFunction")]
public static void Run(
    [TimerTrigger("0 */5 * * * *")] TimerInfo timer,
    FunctionContext context)
{
    context.GetLogger("Timer").LogInformation($"Last: {timer.ScheduleStatus?.Last}");
}
```

## Event Grid

```csharp
// Trigger
[Function("EventGridTrigger")]
public static void Run(
    [EventGridTrigger] EventGridEvent eventGridEvent,
    FunctionContext context)
{
    context.GetLogger("EG").LogInformation($"Event: {eventGridEvent.Subject}");
}

// Output
[EventGridOutput(TopicEndpointUri = "MyTopicUri", TopicKeySetting = "MyTopicKey")]
```

## Cosmos DB

```csharp
// Trigger (Change Feed)
[Function("CosmosDBTrigger")]
public static void Run(
    [CosmosDBTrigger("mydb", "mycontainer",
     Connection = "CosmosDBConnection",
     CreateLeaseContainerIfNotExists = true)] IReadOnlyList<MyDocument> documents,
    FunctionContext context)
{
    foreach (var doc in documents)
        context.GetLogger("Cosmos").LogInformation($"Doc: {doc.Id}");
}

// Input
[CosmosDBInput("mydb", "mycontainer", Connection = "CosmosDBConnection",
 Id = "{id}", PartitionKey = "{partitionKey}")] MyDocument doc

// Output
[CosmosDBOutput("mydb", "mycontainer", Connection = "CosmosDBConnection")]
```

## Service Bus

```csharp
// Queue Trigger
[Function("SBQueueTrigger")]
public static void Run(
    [ServiceBusTrigger("myqueue", Connection = "ServiceBusConnection")] string message,
    FunctionContext context)
{
    context.GetLogger("SB").LogInformation($"Message: {message}");
}

// Topic Trigger
[Function("SBTopicTrigger")]
public static void Run(
    [ServiceBusTrigger("mytopic", "mysubscription",
     Connection = "ServiceBusConnection")] string message,
    FunctionContext context)
{
    context.GetLogger("SB").LogInformation($"Topic: {message}");
}

// Output
[ServiceBusOutput("outqueue", Connection = "ServiceBusConnection")]
```

## Event Hubs

```csharp
// Trigger
[Function("EventHubTrigger")]
public static void Run(
    [EventHubTrigger("myeventhub", Connection = "EventHubConnection")] EventData[] events,
    FunctionContext context)
{
    foreach (var e in events)
        context.GetLogger("EH").LogInformation($"Event: {e.EventBody}");
}

// Output
[EventHubOutput("outeventhub", Connection = "EventHubConnection")]
```

## Table Storage

```csharp
// Input
[TableInput("mytable", "{partitionKey}", "{rowKey}",
 Connection = "AzureWebJobsStorage")] TableEntity entity

// Output
[TableOutput("mytable", Connection = "AzureWebJobsStorage")]
```

## SQL

```csharp
// Trigger
[Function("SqlTrigger")]
public static void Run(
    [SqlTrigger("dbo.MyTable", "SqlConnection")] IReadOnlyList<SqlChange<MyItem>> changes,
    FunctionContext context)
{
    foreach (var change in changes)
        context.GetLogger("SQL").LogInformation($"Change: {change.Item.Id}");
}

// Input
[SqlInput("SELECT * FROM dbo.MyTable WHERE Id = @Id",
 "SqlConnection", parameters: "@Id={id}")] IEnumerable<MyItem> items

// Output
[SqlOutput("dbo.MyTable", "SqlConnection")]
```

## SignalR

```csharp
// Output
[SignalROutput(HubName = "myhub", ConnectionStringSetting = "AzureSignalRConnectionString")]
```

## SendGrid

```csharp
[SendGridOutput(ApiKey = "SendGridApiKey", From = "noreply@example.com")]
```

## Multiple Outputs

```csharp
// Use a custom return type for multiple outputs
public class MultiOutput
{
    [QueueOutput("outqueue", Connection = "AzureWebJobsStorage")]
    public string QueueMessage { get; set; }

    public HttpResponseData HttpResponse { get; set; }
}

[Function("MultiOutput")]
public static MultiOutput Run(
    [HttpTrigger(AuthorizationLevel.Anonymous, "post")] HttpRequestData req)
{
    return new MultiOutput
    {
        QueueMessage = "new item",
        HttpResponse = req.CreateResponse(HttpStatusCode.OK)
    };
}
```

> Full reference: [Azure Functions C# isolated worker guide](https://learn.microsoft.com/en-us/azure/azure-functions/dotnet-isolated-process-guide)
