# TypeScript — Azure Functions v4 Triggers & Bindings

> **Model**: Node.js v4 programming model (TypeScript). Same as JavaScript v4 with type annotations.
> **NO** `function.json` files.
> Import: `import { app, HttpRequest, HttpResponseInit, InvocationContext, input, output } from '@azure/functions';`

TypeScript uses the same `app.*()` registration as JavaScript. See [javascript.md](javascript.md) for all trigger/binding patterns. Below are TypeScript-specific type signatures.

## Cloud Run Functions Migration Rules

> Full scenario guidance → [cloudrun-functions-to-functions.md](../cloudrun-functions-to-functions.md). Canonical side-by-side code examples → [javascript.md — Cloud Run Functions Migration Rules](javascript.md#cloud-run-functions-migration-rules) (the runtime is identical; only the type imports change).

TypeScript-specific notes when migrating Functions Framework code:

- **Drop the `@google-cloud/functions-framework` types** in `tsconfig` / `package.json`:
  - Remove `import { HttpFunction, CloudEventFunction, Request, Response } from '@google-cloud/functions-framework';`
  - Remove `import { CloudEvent } from 'cloudevents';`
- **Import Azure Functions types** instead: `import { app, HttpRequest, HttpResponseInit, InvocationContext } from '@azure/functions';`
- **Map handler types**:

  | Functions Framework type | Azure Functions equivalent |
  |--------------------------|---------------------------|
  | `HttpFunction` (= `(req: Request, res: Response) => void \| Promise<void>`) | `(request: HttpRequest, context: InvocationContext) => Promise<HttpResponseInit>` |
  | `CloudEventFunction<T>` (= `(cloudEvent: CloudEvent<T>) => void \| Promise<void>`) | Trigger-specific: e.g., `(message: T, context: InvocationContext) => Promise<void>` for Service Bus; `(blob: Buffer, context: InvocationContext) => Promise<void>` for Blob |
- **Drop `cloudevents` package** entirely once handlers are converted

### TypeScript HTTP Handler Skeleton

```typescript
// ❌ BEFORE — Cloud Run functions (TypeScript)
import { http, Request, Response } from '@google-cloud/functions-framework';

http('helloHttp', (req: Request, res: Response) => {
  res.status(200).send(`Hello ${req.query.name ?? 'World'}`);
});

// ✅ AFTER — Azure Functions v4 (TypeScript)
import { app, HttpRequest, HttpResponseInit, InvocationContext } from '@azure/functions';

app.http('helloHttp', {
  methods: ['GET', 'POST'],
  authLevel: 'anonymous',
  handler: async (request: HttpRequest, context: InvocationContext): Promise<HttpResponseInit> => {
    const name = request.query.get('name') ?? 'World';
    return { status: 200, body: `Hello ${name}` };
  }
});
```

## HTTP Trigger

```typescript
import { app, HttpRequest, HttpResponseInit, InvocationContext } from '@azure/functions';

app.http('httpFunction', {
  methods: ['GET', 'POST'],
  authLevel: 'anonymous',
  handler: async (request: HttpRequest, context: InvocationContext): Promise<HttpResponseInit> => {
    const name = request.query.get('name') || (await request.text());
    return { body: `Hello, ${name}!` };
  }
});
```

## Blob Storage Trigger

```typescript
import { app, InvocationContext } from '@azure/functions';

app.storageBlob('blobTrigger', {
  path: 'samples-workitems/{name}',
  connection: 'AzureWebJobsStorage',
  source: 'EventGrid',
  handler: async (blob: Buffer, context: InvocationContext): Promise<void> => {
    context.log(`Blob: ${context.triggerMetadata.name}, Size: ${blob.length}`);
  }
});
```

## Queue Storage Trigger

```typescript
app.storageQueue('queueTrigger', {
  queueName: 'myqueue-items',
  connection: 'AzureWebJobsStorage',
  handler: async (queueItem: unknown, context: InvocationContext): Promise<void> => {
    context.log('Queue item:', queueItem);
  }
});
```

## Timer Trigger

```typescript
import { app, InvocationContext, Timer } from '@azure/functions';

app.timer('timerFunction', {
  schedule: '0 */5 * * * *',
  handler: async (myTimer: Timer, context: InvocationContext): Promise<void> => {
    context.log('Timer fired at:', myTimer.scheduleStatus?.last);
  }
});
```

## Cosmos DB Trigger

```typescript
app.cosmosDB('cosmosDBTrigger', {
  connectionStringSetting: 'CosmosDBConnection',
  databaseName: 'mydb',
  containerName: 'mycontainer',
  createLeaseContainerIfNotExists: true,
  handler: async (documents: unknown[], context: InvocationContext): Promise<void> => {
    documents.forEach(doc => context.log('Changed doc:', doc));
  }
});
```

## Service Bus Queue Trigger

```typescript
app.serviceBusQueue('sbQueueTrigger', {
  queueName: 'myqueue',
  connection: 'ServiceBusConnection',
  handler: async (message: unknown, context: InvocationContext): Promise<void> => {
    context.log('Message:', message);
  }
});
```

## Event Grid Trigger

```typescript
import { app, EventGridEvent, InvocationContext } from '@azure/functions';

app.eventGrid('eventGridTrigger', {
  handler: async (event: EventGridEvent, context: InvocationContext): Promise<void> => {
    context.log('Event:', event.subject, event.eventType);
  }
});
```

## Event Hubs Trigger

```typescript
app.eventHub('eventHubTrigger', {
  eventHubName: 'myeventhub',
  connection: 'EventHubConnection',
  cardinality: 'many',
  handler: async (events: unknown[], context: InvocationContext): Promise<void> => {
    events.forEach(event => context.log('Event:', event));
  }
});
```

## Input/Output Bindings

TypeScript uses the same `input.*()` and `output.*()` helpers as JavaScript. See [javascript.md](javascript.md) for full binding examples — all patterns are identical with added type annotations.

> Full reference: [Azure Functions TypeScript developer guide](https://learn.microsoft.com/en-us/azure/azure-functions/functions-reference-node?tabs=typescript)
