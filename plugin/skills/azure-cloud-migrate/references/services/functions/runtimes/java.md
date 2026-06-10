# Java — Azure Functions Triggers & Bindings

## Cloud Run Functions Migration Rules

> Full scenario guidance → [cloudrun-functions-to-functions.md](../cloudrun-functions-to-functions.md)

Java-specific Cloud Run functions rules:
- **Drop `com.google.cloud.functions:functions-framework-api`** dependency from `pom.xml`
- **Replace `HttpFunction` / `CloudEventsFunction` / `BackgroundFunction` interfaces** with Azure Functions Java method annotations (`@HttpTrigger`, `@ServiceBusQueueTrigger`, `@BlobTrigger`, `@TimerTrigger`)
- **HTTP signature change**:
  - GCP: `void service(HttpRequest req, HttpResponse res) throws Exception` (writes via `res.getWriter()`)
  - Azure: `HttpResponseMessage run(@HttpTrigger HttpRequestMessage<Optional<String>> req, ExecutionContext context)` (returns response)
- **CloudEvent unwrap**:
  - GCP: `void accept(CloudEvent event) throws Exception` (`event.getData().toBytes()` requires JSON parsing for Pub/Sub `MessagePublishedData`)
  - Azure: trigger receives the unwrapped payload directly (`@ServiceBusQueueTrigger String message`, `@BlobTrigger byte[] content`)
- **Drop Buildpacks `pom.xml` profile** (the `function-maven-plugin` invocation) — replace with `azure-functions-maven-plugin`

### Canonical Side-by-Side — HTTP Function

```java
// ❌ BEFORE — Cloud Run functions (Java)
import com.google.cloud.functions.HttpFunction;
import com.google.cloud.functions.HttpRequest;
import com.google.cloud.functions.HttpResponse;

public class HelloHttp implements HttpFunction {
  @Override
  public void service(HttpRequest req, HttpResponse res) throws Exception {
    String name = req.getFirstQueryParameter("name").orElse("World");
    res.getWriter().write("Hello, " + name + "!");
  }
}

// ✅ AFTER — Azure Functions (Java)
import com.microsoft.azure.functions.*;
import com.microsoft.azure.functions.annotation.*;

public class HelloHttp {
  @FunctionName("helloHttp")
  public HttpResponseMessage run(
      @HttpTrigger(name = "req",
                   methods = {HttpMethod.GET, HttpMethod.POST},
                   authLevel = AuthorizationLevel.ANONYMOUS) HttpRequestMessage<Optional<String>> req,
      final ExecutionContext context) {
    String name = req.getQueryParameters().getOrDefault("name", "World");
    return req.createResponseBuilder(HttpStatus.OK).body("Hello, " + name + "!").build();
  }
}
```

### Canonical Side-by-Side — Pub/Sub CloudEvent → Service Bus Queue

```java
// ❌ BEFORE — Cloud Run functions consuming Pub/Sub
import com.google.cloud.functions.CloudEventsFunction;
import io.cloudevents.CloudEvent;

public class HandleMessage implements CloudEventsFunction {
  @Override
  public void accept(CloudEvent event) throws Exception {
    // event.getData() is a MessagePublishedData JSON; data is base64 inside
    byte[] body = event.getData().toBytes();   // requires JSON parse + base64 decode
    // ...business logic...
  }
}

// ✅ AFTER — Azure Functions Service Bus queue trigger
public class HandleMessage {
  @FunctionName("handleMessage")
  public void run(
      @ServiceBusQueueTrigger(name = "msg",
                              queueName = "messages",
                              connection = "ServiceBusConnection") String message,
      final ExecutionContext context) {
    // message is the already-decoded payload
    context.getLogger().info("Got message: " + message);
  }
}
```


> **Model**: Java annotation-based model. Uses `@FunctionName` and trigger/binding annotations.
> Import: `com.microsoft.azure.functions.*` and `com.microsoft.azure.functions.annotation.*`

## HTTP Trigger

```java
@FunctionName("HttpFunction")
public HttpResponseMessage run(
    @HttpTrigger(name = "req", methods = {HttpMethod.GET, HttpMethod.POST},
                 authLevel = AuthorizationLevel.ANONYMOUS) HttpRequestMessage<Optional<String>> request,
    final ExecutionContext context) {
    String name = request.getQueryParameters().get("name");
    return request.createResponseBuilder(HttpStatus.OK).body("Hello, " + name).build();
}
```

## Blob Storage

```java
// Trigger
@FunctionName("BlobTrigger")
public void run(
    @BlobTrigger(name = "blob", path = "samples-workitems/{name}",
                 connection = "AzureWebJobsStorage") byte[] content,
    @BindingName("name") String name,
    final ExecutionContext context) {
    context.getLogger().info("Blob: " + name + ", Size: " + content.length);
}

// Input
@BlobInput(name = "inputBlob", path = "input/{name}", connection = "AzureWebJobsStorage")

// Output
@BlobOutput(name = "outputBlob", path = "output/{name}-out", connection = "AzureWebJobsStorage")
```

## Queue Storage

```java
// Trigger
@FunctionName("QueueTrigger")
public void run(
    @QueueTrigger(name = "message", queueName = "myqueue-items",
                  connection = "AzureWebJobsStorage") String message,
    final ExecutionContext context) {
    context.getLogger().info("Queue message: " + message);
}

// Output
@QueueOutput(name = "output", queueName = "outqueue", connection = "AzureWebJobsStorage")
```

## Timer

```java
@FunctionName("TimerFunction")
public void run(
    @TimerTrigger(name = "timer", schedule = "0 */5 * * * *") String timerInfo,
    final ExecutionContext context) {
    context.getLogger().info("Timer triggered");
}
```

## Event Grid

```java
// Trigger
@FunctionName("EventGridTrigger")
public void run(
    @EventGridTrigger(name = "event") EventGridEvent event,
    final ExecutionContext context) {
    context.getLogger().info("Event: " + event.subject());
}

// Output
@EventGridOutput(name = "output", topicEndpointUri = "MyTopicUri",
                 topicKeySetting = "MyTopicKey")
```

## Cosmos DB

```java
// Trigger (Change Feed)
@FunctionName("CosmosDBTrigger")
public void run(
    @CosmosDBTrigger(name = "documents", databaseName = "mydb",
                     containerName = "mycontainer", connection = "CosmosDBConnection",
                     createLeaseContainerIfNotExists = true) String[] documents,
    final ExecutionContext context) {
    for (String doc : documents) context.getLogger().info("Doc: " + doc);
}

// Input
@CosmosDBInput(name = "doc", databaseName = "mydb", containerName = "mycontainer",
               connection = "CosmosDBConnection", id = "{id}", partitionKey = "{pk}")

// Output
@CosmosDBOutput(name = "newdoc", databaseName = "mydb", containerName = "mycontainer",
                connection = "CosmosDBConnection")
```

## Service Bus

```java
// Queue Trigger
@FunctionName("SBQueueTrigger")
public void run(
    @ServiceBusQueueTrigger(name = "message", queueName = "myqueue",
                            connection = "ServiceBusConnection") String message,
    final ExecutionContext context) {
    context.getLogger().info("Message: " + message);
}

// Topic Trigger
@FunctionName("SBTopicTrigger")
public void run(
    @ServiceBusTopicTrigger(name = "message", topicName = "mytopic",
                            subscriptionName = "mysubscription",
                            connection = "ServiceBusConnection") String message,
    final ExecutionContext context) {
    context.getLogger().info("Topic: " + message);
}

// Output
@ServiceBusQueueOutput(name = "output", queueName = "outqueue",
                       connection = "ServiceBusConnection")
```

## Event Hubs

```java
// Trigger
@FunctionName("EventHubTrigger")
public void run(
    @EventHubTrigger(name = "event", eventHubName = "myeventhub",
                     connection = "EventHubConnection", cardinality = Cardinality.MANY) List<String> events,
    final ExecutionContext context) {
    events.forEach(e -> context.getLogger().info("Event: " + e));
}

// Output
@EventHubOutput(name = "output", eventHubName = "outeventhub",
                connection = "EventHubConnection")
```

## Table Storage

```java
// Input
@TableInput(name = "entity", tableName = "mytable", partitionKey = "{pk}",
            rowKey = "{rk}", connection = "AzureWebJobsStorage")

// Output
@TableOutput(name = "output", tableName = "mytable", connection = "AzureWebJobsStorage")
```

## SQL

```java
// Trigger
@FunctionName("SqlTrigger")
public void run(
    @SqlTrigger(name = "changes", tableName = "dbo.MyTable",
                connectionStringSetting = "SqlConnection") SqlChange[] changes,
    final ExecutionContext context) {
    for (SqlChange change : changes) context.getLogger().info("Change: " + change);
}

// Input
@SqlInput(name = "items", commandText = "SELECT * FROM dbo.MyTable WHERE Id = @Id",
          commandType = "Text", parameters = "@Id={id}",
          connectionStringSetting = "SqlConnection")

// Output
@SqlOutput(name = "output", commandText = "dbo.MyTable",
           connectionStringSetting = "SqlConnection")
```

## SignalR

```java
@SignalROutput(name = "signalr", hubName = "myhub",
               connectionStringSetting = "AzureSignalRConnectionString")
```

## SendGrid

```java
@SendGridOutput(name = "mail", apiKey = "SendGridApiKey",
                from = "noreply@example.com", to = "{email}")
```

## Multiple Outputs

Java uses `OutputBinding<T>` for additional outputs:

```java
@FunctionName("MultiOutput")
public HttpResponseMessage run(
    @HttpTrigger(name = "req", methods = {HttpMethod.POST},
                 authLevel = AuthorizationLevel.ANONYMOUS) HttpRequestMessage<String> request,
    @QueueOutput(name = "output", queueName = "outqueue",
                 connection = "AzureWebJobsStorage") OutputBinding<String> queueOutput,
    final ExecutionContext context) {
    queueOutput.setValue("new item");
    return request.createResponseBuilder(HttpStatus.OK).body("Processed").build();
}
```

> **Note**: Java uses `function.json` generated at build time by the Maven plugin — you don't write them manually.

> Full reference: [Azure Functions Java developer guide](https://learn.microsoft.com/en-us/azure/azure-functions/functions-reference-java)
