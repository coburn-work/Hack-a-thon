---
applyTo: "**/ARMTemplateParameterFiles/**"
---

# Shamrock Foods ARM Parameter Files Conventions

## The Rule (§16.4)

**Always update ALL FOUR environment parameter files** (`dev`, `qa`, `uat`, `prod`) when making any change. Never update just one environment.

## File Location & Naming (§16.2)

```
{ProjectRoot}/ARMTemplateParameterFiles/
    {ProjectName}.dev.parameters.V2.json
    {ProjectName}.qa.parameters.V2.json
    {ProjectName}.uat.parameters.V2.json
    {ProjectName}.prod.parameters.V2.json
```

## Parameter Sections (§16.3)

| Section | Purpose |
|---|---|
| `parameters.appSettings.value[]` | Plain-text app settings (`name` + `value`) |
| `parameters.protectedAppSettings.value[]` | Key Vault-backed settings (`name` + `secretId`) |
| `parameters.connectionStringInfos.value[]` | Oracle/ADO.NET connections (`databaseName`, `userName`, `passwordSecretName`) |
| `parameters.sqlDatabases.value[]` | Azure SQL databases (`databaseName`, `serverName`, etc.) |
| `eventGridSubscriptions[].subscriptions[]` | Event Grid subscriptions (`name`, `endpoint`, `filter.eventTypes[]`) |

## Adding App Settings (§16.5)

Add to `parameters.appSettings.value[]` in all four files:

```json
{ "name": "SettingName", "value": "value" }
```

For Key Vault-backed settings, add to `protectedAppSettings` with environment-specific `secretId`.

## Adding Event Grid Subscriptions (§16.6)

Add to `eventGridSubscriptions[0].subscriptions[]` with environment-specific naming:

```json
{
  "name": "{service}-{subscription}-{env}-egs",
  "endpoint": {
    "endpointType": "WebHook",
    "webhookInfo": { "url": "https://sfc-{service}-{env}-api.azurewebsites.net/events" }
  },
  "filter": { "eventTypes": ["MyEventType"] },
  "retryPolicies": { "maxDeliveryAttempts": 5, "eventTimeToLiveInMinutes": 300 }
}
```

Replace `-dev-` with `-qa-`, `-uat-`, `-prod-` in the other three files.

## Adding Dead Letter Subscriptions (§16.7)

Dead letter subscriptions use `eventGridSystemSubscriptions[]` to process events that failed delivery to the original subscription. They listen for `Microsoft.Storage.BlobCreated` on the dead letter storage account's system topic.

**Key properties:**
- **`subscriptionName`**: `{service}-{subscription}-{env}-ses` (system event subscription suffix)
- **`systemTopicName`**: `sfcdeadltr01{env}sa-{env}-topic` — the system topic on the dead letter storage account
- **`systemTopicResourceGroupName`**: `sfc-storage-{env}-rg`
- **`systemTopicRegion`**: `westus3`
- **`endpoint`**: WebHook pointing to a dead-letter-queue handler on the consuming service (e.g., `https://sfc-{service}-{env}-api.azurewebsites.net/{path}/dead-letter-queue/{handler}`) with `maxEventsPerBatch: 1`
- **`filter.subjectBeginsWith`**: `/blobServices/default/containers/applicationevents-deadletter/blobs/{original-topic}/{ORIGINAL-SUBSCRIPTION-NAME-UPPERCASED}`
- **`filter.eventTypes`**: `["Microsoft.Storage.BlobCreated"]`
- **`retryPolicies`**: Higher values than standard subscriptions (e.g., `maxDeliveryAttempts: 30`, `eventTimeToLiveInMinutes: 1440`)

Replace `-dev-` / `dev` segments with the appropriate environment in all four parameter files.

## Naming Conventions

- Azure SQL databases: `{service-name}-{env}-db` (e.g., `salesorder-dev-db`)
- Application Insights: shared `sfc-shared01-{env}-ai` — services do not provision their own
- LogicMonitor is the alerting platform (separate from Application Insights log destination)

## Environment Suffixes

Use consistent `-{env}-` segments in resource names, URLs, and secret names:
- `dev` → `qa` → `uat` → `prod`
- Pattern visible in existing parameter files — follow the established convention for each project

## Cross-References

- **ARM Template schemas:** The full ARM template definitions (resource structure, allowed properties, and schema) are maintained at [shamrockfoods/azure-arm-templates](https://github.com/shamrockfoods/azure-arm-templates). Consult this repository when adding new resource types or verifying parameter structure.
