# Cloud Run Functions Assessment Inserts

When the source platform is **GCP Cloud Run functions** (or Cloud Functions 1st/2nd gen), use the GCP-flavored tables below in place of the AWS-flavored tables in [assessment.md](assessment.md). Section numbering matches `assessment.md` so the final report can stitch these inserts in directly.

> Replace section content; keep section headings as defined in `assessment.md` (mandatory).

## 1. Executive Summary (GCP variant)

| Property | Value |
|----------|-------|
| **Total Functions** | <count> |
| **Source Platform** | GCP Cloud Run functions (2nd gen) / Cloud Functions 1st gen |
| **Source Runtime** | <runtime and version, e.g., nodejs20> |
| **Target Platform** | Azure Functions |
| **Target Runtime** | <runtime and version> |
| **Target Hosting Plan** | <Flex Consumption / Premium> |
| **Migration Readiness** | <High / Medium / Low> |
| **Estimated Effort** | <Low / Medium / High> |
| **Assessment Date** | <date> |

## 2. Functions Inventory (GCP variant)

> Use `gcloud functions list --gen2 --regions=<region>` and `gcloud functions list --no-gen2 --regions=<region>` to enumerate. Capture entry point and trigger from `gcloud functions describe <name> --gen2 --region=<region>`.

| # | Function Name | Generation | Runtime | Entry Point | Trigger Type | CloudEvent Type | Memory | Timeout (s) | Concurrency | Min/Max Instances | Description |
|---|--------------|-----------|---------|-------------|--------------|-----------------|--------|-------------|-------------|-------------------|-------------|
| 1 | | 1st / 2nd | | | http / event | `google.cloud....` | 256Mi | 60 | 80 | 0 / 100 | |

## 3. Service Mapping (GCP variant)

| GCP Service | Azure Equivalent | Migration Complexity | Notes |
|-------------|------------------|----------------------|-------|
| Cloud Run functions | Azure Functions | Low | Function-shaped code maps 1:1 |
| Cloud Functions 1st gen | Azure Functions | Low–Medium | Background signatures differ — see scenario file |
| HTTPS Load Balancer + Cloud Run | Azure Front Door + Azure Functions (or APIM) | Medium | |
| Pub/Sub | Service Bus (transactional) / Event Grid (broadcast) | Low | Drop base64 decode of `message.data` |
| Cloud Storage | Azure Blob Storage | Low | Use `source: 'EventGrid'` for triggers |
| Firestore (Native) | Cosmos DB (NoSQL API) | Medium | Map document path → container + partition key |
| Cloud SQL (Postgres / MySQL) | Azure Database for PostgreSQL / MySQL Flexible Server | Low–Medium | Replace SA proxy with Entra ID auth |
| Cloud Spanner | Cosmos DB or Azure SQL | High | Distributed SQL semantics differ |
| Secret Manager | Azure Key Vault | Low | Use Key Vault references in app settings |
| Cloud Scheduler | Timer trigger (NCRONTAB) | Low | Prepend `0 ` to cron expression |
| Cloud Tasks | Service Bus queue + scheduled enqueue | Medium | Map per-task `scheduleTime` to `ScheduledEnqueueTimeUtc` |
| Eventarc | Event Grid | Low | |
| Cloud Build | GitHub Actions / Azure DevOps Pipelines | Low | |
| Artifact Registry | Azure Container Registry (ACR) | Low | Only if container deploy |
| Service Account + JSON key | User-Assigned Managed Identity (no keys) | Low | Security improvement |
| Workload Identity Federation | Federated identity credential on UAMI | Low | Same OIDC pattern |
| Cloud KMS | Azure Key Vault (keys) | Medium | |
| Cloud Logging | Application Insights + Azure Monitor Logs | Low | |
| Cloud Trace | Application Insights distributed tracing | Low | |
| Cloud Monitoring | Azure Monitor metrics + alerts | Low | |

## 4. Trigger & Binding Mapping (GCP variant)

| # | Function Name | GCP Trigger | CloudEvent `type` | Azure Trigger | Azure Bindings (extraInputs/extraOutputs) | Notes |
|---|--------------|-------------|-------------------|---------------|-------------------------------------------|-------|
| 1 | | http / pubsub / gcs / firestore / scheduler | `google.cloud.…` | `app.http` / `app.serviceBusQueue` / `app.storageBlob` / `app.cosmosDB` / `app.timer` | | |

## 5. Dependencies Analysis (GCP variant)

| # | Package/Library | Version | GCP-Specific? | Azure Equivalent | Compatible? | Notes |
|---|----------------|---------|---------------|------------------|-------------|-------|
| 1 | `@google-cloud/functions-framework` | | Yes (runtime adapter) | (remove — replaced by `@azure/functions`) | N/A — drop | |
| 2 | `@google-cloud/storage` | | Yes | `@azure/storage-blob` or storage binding | | Prefer binding |
| 3 | `@google-cloud/pubsub` | | Yes | `@azure/service-bus` or queue binding | | Prefer binding |
| 4 | `@google-cloud/firestore` | | Yes | `@azure/cosmos` or Cosmos binding | | Schema migration may be needed |
| 5 | `@google-cloud/secret-manager` | | Yes | Key Vault references (no SDK needed) | | Auth via UAMI |
| 6 | `@google-cloud/logging` | | Yes | Built-in App Insights via `context.log` | | Drop SDK |
| 7 | `@google-cloud/trace-agent` | | Yes | App Insights auto-instrumentation | | Drop SDK |
| 8 | `google-auth-library` / `googleapis` | | Yes | `@azure/identity` (`DefaultAzureCredential`) | | UAMI, no JSON keys |
| 9 | (3rd-party — Express, Lodash, etc.) | | No | Same package on Azure | ✅ | Carry over |

## 6. Environment Variables & Configuration (GCP variant)

| # | GCP Variable / Flag | Purpose | Azure Equivalent | Auth Method | Notes |
|---|--------------------|---------|------------------|-------------|-------|
| 1 | `--set-env-vars` value | App config | App setting in Bicep | App setting | |
| 2 | `--set-secrets=K=projects/.../secrets/...:latest` | Secret injection | App setting with `@Microsoft.KeyVault(SecretUri=...)` | Managed Identity → Key Vault | |
| 3 | `GOOGLE_APPLICATION_CREDENTIALS` | SA JSON key path | **DROP** — use `DefaultAzureCredential` with UAMI clientId | Managed Identity | Security improvement |
| 4 | `FUNCTION_TARGET`, `FUNCTION_SIGNATURE_TYPE`, `K_SERVICE`, `K_REVISION` | Functions Framework / Knative runtime hints | **DROP** | n/a | Not portable |
| 5 | `PORT` (Buildpacks default 8080) | HTTP listener port | **DROP** — Azure Functions host owns the port | n/a | |

## 8. IAM & Security Mapping (GCP variant)

| GCP IAM Role | Azure RBAC Role | Scope | Notes |
|--------------|-----------------|-------|-------|
| `roles/storage.objectAdmin` | Storage Blob Data Contributor | Storage account | |
| `roles/storage.objectViewer` | Storage Blob Data Reader | Storage account | |
| `roles/pubsub.subscriber` | Azure Service Bus Data Receiver | SB namespace | |
| `roles/pubsub.publisher` | Azure Service Bus Data Sender | SB namespace | |
| `roles/datastore.user` | Cosmos DB Built-in Data Contributor | Cosmos account | |
| `roles/secretmanager.secretAccessor` | Key Vault Secrets User | Key Vault | KV references handle this |
| `roles/cloudsql.client` | Entra ID DB user grant | Postgres/MySQL server | No password |
| `roles/eventarc.eventReceiver` | EventGrid EventSubscription Contributor | Resource group | If dynamically creating subs |
| `roles/run.invoker` (internal-only) | Function auth keys / APIM / Easy Auth | Function | Pattern shifts |

## 9. Monitoring & Observability Mapping (GCP variant)

| GCP Service | Azure Equivalent | Migration Notes |
|-------------|------------------|-----------------|
| Cloud Logging | Application Insights + Azure Monitor Logs (Log Analytics) | `console.log` / `context.log` flows automatically via App Insights |
| Cloud Logging structured logs (`severity`, `jsonPayload`) | App Insights custom dimensions | Map `severity` → `SeverityLevel` |
| Cloud Trace | Application Insights distributed tracing | Auto-instrumented for HTTP/SDK calls |
| Cloud Monitoring metrics | Azure Monitor metrics | |
| Cloud Monitoring alerts | Azure Monitor alerts (action groups) | |
| Cloud Monitoring uptime checks | Application Insights availability tests | |
| Error Reporting | App Insights exceptions (auto-captured) | |

## 10. CI/CD & Deployment Mapping (GCP variant)

| GCP Tool | Azure Equivalent | Notes |
|----------|------------------|-------|
| `gcloud functions deploy --gen2` | `func azure functionapp publish` / `azd up` | `azd` recommended (provisions infra + deploys) |
| `gcloud functions deploy --source=.` (Buildpacks) | Oryx remote build via `func` / `azd` | No Dockerfile needed |
| Container image to Artifact Registry | Image to ACR + Premium/Dedicated container hosting | Only when source needs custom container |
| Cloud Build (`cloudbuild.yaml`) | GitHub Actions / Azure DevOps Pipelines | |
| Cloud Deploy | Azure DevOps multi-stage pipelines / `azd` environments | |
| Workload Identity Federation (GitHub Actions → GCP) | OIDC federated credential on UAMI (GitHub Actions → Azure) | Same trust model, different IDs |

## 11. Project Structure Comparison (GCP variant)

| GCP Cloud Run functions Structure | Azure Functions Structure |
|-----------------------------------|--------------------------|
| `index.js` (Functions Framework entry) | `src/app.js` |
| `package.json` with `@google-cloud/functions-framework` + `"start": "functions-framework"` | `package.json` with `@azure/functions` + `"start": "func start"` |
| `Procfile` (Buildpacks, optional) | (none — `host.json` instead) |
| `Dockerfile` (container deploy only) | (drop unless using custom handler) |
| `.gcloudignore` | `.funcignore` |
| `cloudbuild.yaml` | `.github/workflows/*.yml` or `azure-pipelines.yml` |
| Per-function `--entry-point=<name>` arg | `app.<trigger>('<name>', ...)` registration |
| `req`, `res` (Express-like, HTTP) | `request: HttpRequest`; return `HttpResponseInit` |
| `event` (CloudEvent envelope) | Already-unwrapped trigger param (e.g., `message`, `blob`, `event`) |
| `context` (EventContext / no `context` for HTTP) | `context: InvocationContext` (always) |

## 12. Recommendations (Cloud Run functions defaults)

1. **Runtime**: Use the newest GA Node.js LTS supported by Azure Functions (Node 22 at time of writing — verify with [supported languages](https://learn.microsoft.com/azure/azure-functions/supported-languages))
2. **Hosting Plan**: **Flex Consumption** (scale-to-zero, configurable `instanceMemoryMB` and `perInstanceConcurrency` — closest match to Cloud Run's behavior)
3. **IaC Strategy**: Bicep with `azd`
4. **Auth Strategy**: User-Assigned Managed Identity for every service-to-service call. **No** GCP SA JSON keys, no `GOOGLE_APPLICATION_CREDENTIALS`
5. **Secrets**: Key Vault with Key Vault references in app settings (no secrets in env vars directly)
6. **Monitoring**: Application Insights (auto-instrumented) + Azure Monitor
7. **Concurrency**: If source set `--concurrency=80`, set `perInstanceConcurrency: 80` in Flex `functionAppConfig`. Otherwise accept the Flex default (16 HTTP)
8. **Cold start mitigation**: If source set `--min-instances=N`, set `alwaysReady` for the matching trigger group with `instanceCount: N`
