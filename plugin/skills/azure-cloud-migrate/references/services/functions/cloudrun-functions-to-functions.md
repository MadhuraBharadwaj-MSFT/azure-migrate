# GCP Cloud Run Functions to Azure Functions Migration

Detailed guidance for migrating **Google Cloud Run functions** (the 2024 rebrand of **Cloud Functions 2nd gen**, built on Cloud Run + Eventarc) to Azure Functions. Most of this guide also applies to **Cloud Functions 1st gen** — see [1st-Gen Cloud Functions Signature Delta](#1st-gen-cloud-functions-signature-delta) for the small differences.

> **Naming clarification**: In Aug 2024 Google merged Cloud Functions 2nd gen into the Cloud Run family and rebranded the product as **"Cloud Run functions"**. The code (Functions Framework handlers) and the developer experience (`gcloud functions deploy --gen2`) were unchanged. Throughout this document, "Cloud Run functions" refers to that function-shaped product, **not** arbitrary Cloud Run *services*. For full Cloud Run *services* (Express/Flask/Spring HTTP servers), recommend Azure Container Apps instead — see [SKILL.md scenario note](../../../SKILL.md#migration-scenarios).

## Overview

| GCP Service | Azure Equivalent |
|-------------|------------------|
| Cloud Run functions (Cloud Functions 2nd gen) | Azure Functions |
| Cloud Functions 1st gen | Azure Functions |
| Functions Framework (`@google-cloud/functions-framework`) | `@azure/functions` (v4 programming model) |
| Cloud Pub/Sub | Azure Service Bus (queues/topics) **or** Azure Event Grid |
| Cloud Storage (GCS) | Azure Blob Storage |
| GCS `object.finalized` event | Azure Blob Storage + Event Grid (`source: 'EventGrid'`) |
| Firestore (Native mode) | Azure Cosmos DB (NoSQL API) |
| Firestore document events | Cosmos DB Change Feed trigger |
| Cloud SQL (Postgres/MySQL) | Azure Database for PostgreSQL Flexible Server / Azure Database for MySQL Flexible Server |
| Cloud Spanner | Azure Cosmos DB / Azure SQL |
| Secret Manager | Azure Key Vault (with Key Vault references in app settings) |
| Cloud Scheduler | Azure Functions Timer Trigger (NCRONTAB) |
| Cloud Tasks | Azure Service Bus queue (with scheduled enqueue) |
| Eventarc | Azure Event Grid |
| Cloud Audit Logs → Eventarc | Activity Log + Event Grid system topic |
| Cloud Logging | Application Insights + Azure Monitor Logs |
| Cloud Trace | Application Insights (distributed tracing) |
| Cloud Monitoring | Azure Monitor (metrics + alerts) |
| Cloud Build | GitHub Actions / Azure DevOps Pipelines |
| Artifact Registry | Azure Container Registry (ACR) |
| Buildpacks (source deploy) | Oryx (when using `func azure functionapp publish` with source) |
| GCP Service Account (with JSON key) | User-Assigned Managed Identity (UAMI) — **no key files** |
| Workload Identity Federation | Managed Identity + federated credentials (already native on Azure) |
| IAM roles | Azure RBAC |
| Cloud KMS | Azure Key Vault (keys) |

## Programming Model Mapping

> **Core insight**: Cloud Run functions are written using the open-source [**Functions Framework**](https://github.com/GoogleCloudPlatform/functions-framework). Two signatures only: **HTTP function** `(req, res)` and **CloudEvent function** `(cloudEvent)`. Both map cleanly to Azure Functions handlers — **no HTTP-server decomposition is required**.

| Cloud Run functions (Functions Framework) | Azure Functions |
|------------------------------------------|-----------------|
| `exports.<name> = (req, res) => {...}` (HTTP) | `app.http('<name>', { handler })` (v4 JS / TS) |
| `functions.http('<name>', (req, res) => {...})` | `app.http('<name>', { handler })` |
| `functions.cloudEvent('<name>', (event) => {...})` | Event-shape-specific trigger: `app.storageBlob()`, `app.serviceBusQueue()`, `app.eventGrid()`, etc. |
| `req` (Express-like) | `request: HttpRequest` |
| `res` (Express-like) — `res.send()`, `res.status()` | **Return** an `HttpResponseInit` object (`{ status, body, headers }`) |
| `event` (CloudEvent envelope with `data`) | Trigger-typed parameter (`message`, `blob`, `event`) — already-unwrapped data |
| `--target=<name>` CLI flag | First arg of `app.<trigger>('<name>', ...)` |
| `--signature-type=cloudevent` | Implicit from the trigger function used |
| `--entry-point=<name>` (gcloud) | First arg of `app.<trigger>` registration |
| `package.json` `"main"` + `"start": "functions-framework"` | `package.json` `"main": "src/app.js"` + `@azure/functions` import |
| `Promise`-returning handler | `async` handler returning a value/HttpResponseInit |

> ❌ **Drop** the `@google-cloud/functions-framework` dependency entirely. The framework is the Cloud Run functions runtime adapter — it has no role on Azure Functions.

## Trigger / CloudEvent Mapping

The Functions Framework `cloudEvent()` handler receives a CloudEvents 1.0 envelope with a typed `data` payload. Azure Functions triggers consume the *unwrapped* payload directly. Map by **CloudEvent `type`**:

| Cloud Run functions trigger (`type`) | Source service | Azure Functions trigger | Notes |
|--------------------------------------|----------------|------------------------|-------|
| HTTP request (no CloudEvent) | Direct invocation / Eventarc / Cloud Scheduler HTTP | `app.http()` | Direct equivalent. For Scheduler, prefer Timer trigger below. |
| `google.cloud.storage.object.v1.finalized` | Cloud Storage (object created) | `app.storageBlob()` with `source: 'EventGrid'` | See [global-rules.md](global-rules.md) for Flex Consumption + EventGrid source infrastructure requirements (always-ready instance, queue endpoint, Event Grid subscription via Bicep). |
| `google.cloud.storage.object.v1.deleted` | Cloud Storage (object deleted) | Event Grid trigger filtering on `Microsoft.Storage.BlobDeleted` | No native Azure Functions blob "deleted" trigger; use Event Grid. |
| `google.cloud.storage.object.v1.metadataUpdated` | Cloud Storage | Event Grid + filter | Use Event Grid system topic on the storage account. |
| `google.cloud.storage.object.v1.archived` | Cloud Storage | Event Grid + custom logic | No direct equivalent (Azure tiering is policy-based, not event-based). |
| `google.cloud.pubsub.topic.v1.messagePublished` | Pub/Sub | `app.serviceBusQueue()` (default) **or** `app.eventGrid()` | Choose Service Bus for transactional messaging (sessions, dead-letter, FIFO). Choose Event Grid for fan-out/event-broadcast semantics. **Base64-decode `event.data.message.data` first** — Pub/Sub payloads are base64; Service Bus messages are raw. |
| `google.cloud.firestore.document.v1.created` | Firestore (Native) | Cosmos DB Change Feed trigger (`app.cosmosDB()`) | Change Feed surfaces inserts and updates — filter by `_ts` if you need create-only semantics. |
| `google.cloud.firestore.document.v1.updated` | Firestore | Cosmos DB Change Feed | Same trigger; Change Feed emits both. |
| `google.cloud.firestore.document.v1.deleted` | Firestore | Cosmos DB Change Feed with **soft-delete** pattern, OR Event Grid via Activity Log | Cosmos Change Feed does not emit deletes natively. |
| `google.cloud.audit.log.v1.written` | Cloud Audit Logs (via Eventarc) | Azure Activity Log → Event Grid system topic → `app.eventGrid()` | Filter on `operationName` to match GCP `methodName`. |
| Cloud Scheduler → HTTP target | Cloud Scheduler | `app.timer()` with NCRONTAB | Cloud Scheduler uses standard cron (unix); NCRONTAB has an extra seconds field — **prepend `0 `** to convert. |
| Cloud Tasks → HTTP target | Cloud Tasks | `app.serviceBusQueue()` with **scheduled enqueue time** | Use `ServiceBusMessage.ScheduledEnqueueTimeUtc` for Cloud Tasks `scheduleTime` equivalent. |
| `google.cloud.scheduler.job.v1.executed` | Cloud Scheduler audit | Activity Log filter, rarely needed | Direct Timer trigger is the right answer in almost all cases. |

> ⚠️ **Pub/Sub payload unwrap**: GCP CloudEvent body for Pub/Sub is `{ message: { data: "<base64>", attributes: {...}, messageId, publishTime }, subscription }`. After migration to Service Bus, the message body is the **already-decoded** payload, and `attributes` becomes `applicationProperties`. Strip the `message.data` base64 decode logic from migrated code.

## Runtime-Specific Migration Patterns

For language-specific side-by-side migration code (Functions Framework → Azure Functions), see the runtime reference for the target language:

| Runtime | Migration Patterns |
|---------|-------------------|
| JavaScript (Node.js v4) | [runtimes/javascript.md — Cloud Run Functions Migration Rules](runtimes/javascript.md#cloud-run-functions-migration-rules) |
| TypeScript (v4) | [runtimes/typescript.md — Cloud Run Functions Migration Rules](runtimes/typescript.md#cloud-run-functions-migration-rules) |
| Python (v2) | [runtimes/python.md — Cloud Run Functions Migration Rules](runtimes/python.md#cloud-run-functions-migration-rules) |
| Java | [runtimes/java.md — Cloud Run Functions Migration Rules](runtimes/java.md#cloud-run-functions-migration-rules) |
| C# (Isolated Worker) | [runtimes/csharp.md — Cloud Run Functions Migration Rules](runtimes/csharp.md#cloud-run-functions-migration-rules) |

> **PowerShell**: GCP does not support PowerShell for Cloud Run functions. If the source is PowerShell, the workload is not Cloud Run functions and this scenario does not apply.

> **Go, Ruby, PHP**: Google supports these in Cloud Run functions, but **Azure Functions does not have a first-party Go or Ruby worker**. Recommend re-platforming to a supported Azure Functions language **or** using an Azure Functions [custom handler](https://learn.microsoft.com/azure/azure-functions/functions-custom-handlers) that wraps the existing binary. Flag this in the assessment report.

## Project Structure

```
REQUIRED for Azure Functions:
src/
├── app.js (or function_app.py)   # Main entry point
├── host.json                      # Function host configuration
├── local.settings.json            # Local development settings
├── package.json (or requirements.txt)
├── [helper-modules]               # Business logic carried over from GCP source
└── tests/                         # Test files

DROP these from the GCP source:
├── index.js (Functions Framework entry)  → merge handlers into src/app.js
├── package.json: "@google-cloud/functions-framework"  → remove
├── package.json: "start": "functions-framework --target=..." → replace with "start": "func start"
├── Procfile (if used by Buildpacks)      → remove
├── Dockerfile (if container deploy)      → drop unless using Azure Functions custom handler
├── .gcloudignore                          → replace with .funcignore
└── cloudbuild.yaml                        → replace with GitHub Actions / ADO YAML

❌ NEVER create:
├── [functionName]/                # No individual function directories
│   ├── function.json              # No function.json (JS v4, Python v2)
│   └── index.js
```

## Environment Variables & Secrets

### gcloud → Azure Functions app settings mapping

| GCP source | Azure Functions equivalent |
|-----------|---------------------------|
| `--set-env-vars=KEY=value` | App setting `KEY` in Bicep `appSettings` |
| `--set-secrets=KEY=projects/<p>/secrets/<s>:latest` | Key Vault reference: `@Microsoft.KeyVault(SecretUri=https://<vault>.vault.azure.net/secrets/<s>/)` |
| `--service-account=<sa>@<p>.iam.gserviceaccount.com` | User-Assigned Managed Identity (UAMI) bound to the function app |
| Implicit `K_SERVICE`, `K_REVISION`, `K_CONFIGURATION` (Knative env) | Use `WEBSITE_SITE_NAME` / `WEBSITE_INSTANCE_ID` if needed; usually safe to remove. |
| `FUNCTION_TARGET`, `FUNCTION_SIGNATURE_TYPE` | Drop — these are Functions Framework runtime hints, not portable. |
| `GOOGLE_APPLICATION_CREDENTIALS` (path to SA JSON) | **Remove entirely**. Use `DefaultAzureCredential` with UAMI client ID instead — see [global-rules.md](global-rules.md#identity-first-authentication-zero-api-keys). |

### Identity-first auth (no JSON key files)

GCP service account JSON keys are a common security anti-pattern that Azure Functions eliminates. **Remove every reference** to `GOOGLE_APPLICATION_CREDENTIALS`, mounted key files, and `google.auth.fromJSON()`.

```javascript
// ❌ DROP from GCP code
const {Storage} = require('@google-cloud/storage');
const storage = new Storage({ keyFilename: '/secrets/sa-key.json' });

// ✅ Azure equivalent — UAMI, no keys
const { DefaultAzureCredential } = require('@azure/identity');
const { BlobServiceClient } = require('@azure/storage-blob');

const credential = new DefaultAzureCredential({
  managedIdentityClientId: process.env.AZURE_CLIENT_ID
});
const blobService = new BlobServiceClient(
  `https://${process.env.STORAGE_ACCOUNT_NAME}.blob.core.windows.net`,
  credential
);
```

> Prefer **bindings** over the SDK shown above when possible — `input.storageBlob()` / `output.storageBlob()` remove credential plumbing entirely. See [global-rules.md](global-rules.md).

## `gcloud functions deploy` Flag → Azure Equivalent

When parsing the user's deployment commands or CI scripts, map each flag to its Azure target:

| `gcloud functions deploy` flag | Azure Functions target | Where it lives |
|-------------------------------|------------------------|----------------|
| `--gen2` | (implicit — Azure Functions is always the latest model) | n/a |
| `--region=<r>` | Azure region for resource group / function app | Bicep `location` parameter |
| `--runtime=nodejs20` | Function app `linuxFxVersion: 'NODE\|20'` (Flex: `runtime.name=node`, `runtime.version=20`) | `infra/main.bicep` |
| `--entry-point=<name>` | First arg of `app.http('<name>', ...)` etc. | `src/app.js` |
| `--source=.` | Project root | `azure.yaml` (azd) `services.api.project` |
| `--trigger-http` | `app.http()` | `src/app.js` |
| `--trigger-topic=<topic>` | Service Bus queue/topic binding | `app.serviceBusQueue()` / `app.serviceBusTopic()` |
| `--trigger-bucket=<bucket>` | Blob trigger with EventGrid source + Event Grid subscription on storage account | `app.storageBlob({ source: 'EventGrid' })` + Bicep Event Grid system topic — see [global-rules.md](global-rules.md#flex-consumption-specifics) |
| `--trigger-event-filters=type=<eventType>` | Event Grid subscription `filter.includedEventTypes` | Bicep `Microsoft.EventGrid/systemTopics/eventSubscriptions` |
| `--memory=512Mi` | `instanceMemoryMB` (Flex Consumption) or App Service Plan SKU | `functionAppConfig.scaleAndConcurrency.instanceMemoryMB` |
| `--timeout=300` | `host.json` `functionTimeout` (max 230s on Consumption, unbounded on Flex/Premium) | `host.json` |
| `--concurrency=80` | Per-instance HTTP concurrency for Flex | `functionAppConfig.scaleAndConcurrency.triggers.http.perInstanceConcurrency` |
| `--min-instances=1` | `alwaysReady: [{ name: '<group>', instanceCount: 1 }]` | Flex `scaleAndConcurrency.alwaysReady` |
| `--max-instances=100` | `maximumInstanceCount` | Flex `scaleAndConcurrency.maximumInstanceCount` |
| `--set-env-vars=K=V` | App setting | Bicep `appSettings` |
| `--set-secrets=K=...:latest` | Key Vault reference in app setting | Bicep `appSettings` with `@Microsoft.KeyVault(...)` |
| `--service-account=<sa>` | User-Assigned Managed Identity | Bicep `Microsoft.ManagedIdentity/userAssignedIdentities` + role assignments |
| `--ingress-settings=internal-only` | Function app `publicNetworkAccess: 'Disabled'` + private endpoint | Bicep `Microsoft.Web/sites` `properties.publicNetworkAccess` |
| `--vpc-connector=<c>` | VNet integration | Bicep `Microsoft.Web/sites/networkConfig` |

> ⚠️ **Concurrency model difference**: Cloud Run functions default to **80 concurrent requests per instance**. Azure Functions on Flex Consumption defaults to **16 per instance for HTTP** (`perInstanceConcurrency`). If the source explicitly relied on per-instance concurrency (e.g., shared in-process caches), call this out in the assessment — either raise `perInstanceConcurrency` to match, or note the behavioral change.

## Buildpacks / Source-Based Deploy → Azure

GCP supports source-based deploy (`gcloud functions deploy --source=.`) where Google's Buildpacks containerize your code automatically. Two Azure equivalents:

| GCP path | Azure equivalent |
|----------|------------------|
| `gcloud functions deploy --source=.` (Buildpacks) | `func azure functionapp publish <app>` (Oryx builds remotely) **or** `azd up` (recommended — provisions infra + deploys) |
| `gcloud functions deploy --source=. --docker-registry=artifact-registry` (containerized) | Dockerfile + ACR build + Flex Consumption container image (Premium/Dedicated for true container hosting; Flex code-based works for most cases) |

Prefer the code-based path (no Dockerfile) on Flex Consumption unless the source has native binary dependencies that justify containerization.

## 1st-Gen Cloud Functions Signature Delta

Cloud Functions 1st gen used non-CloudEvent signatures for background functions. If the source is 1st gen, the **HTTP signature is identical**, but background signatures differ:

| 1st-gen signature | What it gets | Migration |
|------------------|--------------|-----------|
| `exports.<name> = (req, res) => {...}` (HTTP) | Express req/res | Same as 2nd gen — `app.http()` |
| `exports.<name> = (data, context) => {...}` (background) | Raw event `data` + `EventContext` (no CloudEvent envelope) | Migrate to the appropriate Azure trigger (`app.storageBlob`, `app.serviceBusQueue`, etc.) — the `data` shape is the legacy unwrapped event, NOT a CloudEvent envelope. |
| `exports.<name> = (message, context) => {...}` (Pub/Sub legacy) | `{data: "<base64>", attributes: {...}}` | `app.serviceBusQueue()`; remove `Buffer.from(message.data, 'base64').toString()`. |
| `exports.<name> = (object, context) => {...}` (GCS legacy) | GCS object metadata | `app.storageBlob({ source: 'EventGrid' })`. |
| Returning a `Promise` or accepting a `callback` 3rd arg | Either pattern is valid in 1st gen | Use `async` handler returning a value in Azure. |

Tell the user to **`gcloud functions list --gen2 --regions=...`** vs **`gcloud functions list --no-gen2 --regions=...`** to inventory which generation each function uses.

## Auth: GCP Service Account → User-Assigned Managed Identity

| GCP IAM Role | Azure RBAC Role | Scope | Typical use |
|-------------|-----------------|-------|-------------|
| `roles/storage.objectAdmin` | Storage Blob Data Contributor | Storage account / container | Read+write blobs |
| `roles/storage.objectViewer` | Storage Blob Data Reader | Storage account / container | Read-only blobs |
| `roles/pubsub.subscriber` | Azure Service Bus Data Receiver | Service Bus namespace / queue | Receive messages |
| `roles/pubsub.publisher` | Azure Service Bus Data Sender | Service Bus namespace / queue | Send messages |
| `roles/datastore.user` (Firestore) | Cosmos DB Built-in Data Contributor | Cosmos DB account | Read+write documents |
| `roles/secretmanager.secretAccessor` | Key Vault Secrets User | Key Vault | Read secrets (Key Vault references handle this automatically) |
| `roles/cloudsql.client` | (managed-identity DB user grant) | PostgreSQL/MySQL server | Use AAD/Entra auth, not a DB password |
| `roles/eventarc.eventReceiver` | EventGrid EventSubscription Contributor | Resource group | For functions creating EG subscriptions dynamically |
| `roles/cloudtrace.agent` | Monitoring Metrics Publisher | App Insights | Emit traces |
| `roles/logging.logWriter` | (built-in for Functions runtime) | n/a | Implicit via App Insights connection |
| `roles/run.invoker` (HTTPS-only function) | Function-level auth keys + APIM / Entra ID auth | Function | For internal-only invocation, use APIM or Easy Auth |

### Workload Identity Federation → Federated Identity Credentials

If the GCP source uses **Workload Identity Federation** (e.g., from GitHub Actions to GCP without storing SA JSON keys), the Azure equivalent is **Federated Identity Credentials on a User-Assigned Managed Identity** — already a first-class pattern in Azure. No JSON keys, OIDC trust between GitHub and Entra ID. Use `azure/login@v2` with `client-id`, `tenant-id`, `subscription-id` (no `client-secret`).

## Knative `service.yaml` Reference (advanced)

For users who exported `service.yaml` from Cloud Run functions, key fields and their Azure mapping:

```yaml
# GCP Cloud Run service.yaml (relevant subset)
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: "1"        # → alwaysReady[].instanceCount
        autoscaling.knative.dev/maxScale: "100"      # → maximumInstanceCount
        run.googleapis.com/cpu-throttling: "true"    # → (n/a — Flex always throttles between requests)
    spec:
      containerConcurrency: 80                       # → perInstanceConcurrency
      timeoutSeconds: 300                            # → host.json functionTimeout
      serviceAccountName: my-sa@proj.iam...          # → managed identity (UAMI)
      containers:
      - image: gcr.io/proj/img                       # → drop (code-based deploy)
        env:
        - name: KEY
          value: VALUE                               # → appSettings
        - name: SECRET
          valueFrom:
            secretKeyRef:                            # → Key Vault reference
              name: my-secret
              key: latest
        resources:
          limits:
            memory: 512Mi                            # → instanceMemoryMB (round up to 2048/4096)
            cpu: "1"                                 # → (implicit by instanceMemoryMB on Flex)
```

## Reference Links

- [Functions Framework for Node.js](https://github.com/GoogleCloudPlatform/functions-framework-nodejs)
- [Cloud Run functions overview (rebrand)](https://cloud.google.com/blog/products/serverless/google-cloud-functions-is-now-cloud-run-functions)
- [Google CloudEvents types](https://github.com/googleapis/google-cloudevents)
- [Azure Functions supported languages](https://learn.microsoft.com/azure/azure-functions/supported-languages)
- [Azure Functions triggers and bindings overview](https://learn.microsoft.com/azure/azure-functions/functions-triggers-bindings)
- [Azure Functions Flex Consumption hosting](https://learn.microsoft.com/azure/azure-functions/flex-consumption-plan)
- [GCP → Azure services comparison](https://learn.microsoft.com/azure/architecture/gcp-professional/)
- [Functions quickstart JS (azd)](https://github.com/Azure-Samples/functions-quickstart-javascript-azd/tree/main/infra)
- [Azure Functions custom handlers (for unsupported runtimes like Go)](https://learn.microsoft.com/azure/azure-functions/functions-custom-handlers)
