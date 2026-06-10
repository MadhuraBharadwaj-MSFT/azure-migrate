# Workflow Details

## Status Tracking

Maintain a `migration-status.md` file in the output directory (`<workspace-root-basename>-azure/`):

```markdown
# Migration Status
| Phase | Status | Notes |
|-------|--------|-------|
| Assessment | ⬜ Not Started | |
| Code Migration | ⬜ Not Started | |
```

Update status: ⬜ Not Started → 🔄 In Progress → ✅ Complete → ❌ Failed

## User Progress Updates

During long-running operations (Azure deployments, image pushes, environment provisioning), **proactively report progress** so the user is never left waiting without feedback:

1. **Resource-level status table** — After submitting a deployment, poll `az resource list` or `az deployment operation group list` and present a status table:
   ```
   | Resource | Status |
   |----------|--------|
   | VNet     | ✅ Created |
   | ACR      | ✅ Created |
   | Container Apps Env | 🔄 Provisioning |
   | order-service | ⬜ Waiting |
   ```
2. **Explain what's slow** — If a resource takes >2 minutes (e.g., Container Apps Environment with VNet), tell the user *why* ("VNet integration provisions internal load balancers and DNS — this typically takes 3-5 min").
3. **Don't go silent** — If a single `az deployment group create` covers all resources, poll `az resource list -g <rg>` periodically and update the user on newly created resources.
4. **Announce each phase transition** — When moving between skill phases (assess → migrate → deploy → validate), clearly tell the user what just completed and what's next.

## Error Handling

| Error | Cause | Remediation |
|-------|-------|-------------|
| Unsupported runtime | Source runtime not available in target Azure service | Check target service's supported languages documentation |
| Missing service mapping | Source service has no direct Azure equivalent | Use closest Azure alternative, document in assessment |
| Code migration failure | Incompatible patterns or dependencies | Review scenario-specific guide in [lambda-to-functions.md](services/functions/lambda-to-functions.md) or [cloudrun-functions-to-functions.md](services/functions/cloudrun-functions-to-functions.md) |
| `azd init` refuses non-empty directory | azd requires clean directory for template init | Use temp directory approach: init in empty dir, copy files back |
| 1st-gen background function signature detected (GCP) | Source uses `(data, context)` rather than CloudEvent | `data` is the legacy unwrapped event, not a CloudEvent envelope. Migrate Pub/Sub: drop `Buffer.from(message.data, 'base64')`. Migrate GCS: map `object` fields to Azure Blob trigger metadata. See [cloudrun-functions-to-functions.md — 1st-gen Signature Delta](services/functions/cloudrun-functions-to-functions.md#1st-gen-cloud-functions-signature-delta) |
| Eventarc filter has no Azure equivalent | Source filters on a GCP-only event source (Cloud Audit Logs operation, BigQuery, Vertex AI, etc.) | Map to Azure Activity Log + Event Grid system topic where possible; otherwise document as a coverage gap and require app-level polling/webhook |
| Buildpacks-deployed source with no Dockerfile | `gcloud functions deploy --source=.` used Google Buildpacks — no Dockerfile in repo | Expected — Azure Functions remote build (Oryx via `func` / `azd`) handles this. No action needed |
| `@google-cloud/functions-framework` still in package.json after migration | Code-migration phase missed the dependency teardown rule | Remove the dependency and the `functions-framework` script entry. See [code-migration.md — Cloud Run Functions–Specific Code Migration Rules](services/functions/code-migration.md#cloud-run-functionsspecific-code-migration-rules) |
| Pub/Sub message body still base64-decoded after migration | Source `cloudEvent` handler called `Buffer.from(event.data.message.data, 'base64')` | Service Bus messages arrive already decoded — remove the decode step. `event.data.message.attributes` becomes `message.applicationProperties` |
| `GOOGLE_APPLICATION_CREDENTIALS` env var or SA JSON file present in migrated app | Source used a Service Account key file | Remove the env var and the JSON file. Use `DefaultAzureCredential({ managedIdentityClientId })` — see [global-rules.md](services/functions/global-rules.md#identity-first-authentication-zero-api-keys) |

> For scenario-specific errors (e.g., Azure Functions binding issues, trigger configuration), see the error table in the corresponding scenario reference.
