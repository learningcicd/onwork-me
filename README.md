{% raw %}
```yaml


```
{% endraw %}

# monitor-ru.yml — Change Log

This README documents all changes made to `monitor-ru.yml`, an Azure DevOps pipeline for Cosmos DB RU (Request Unit) partition monitoring, from initial transcription through the addition of non-prod support.

## 1. Initial creation

The pipeline was reconstructed from screenshots of the source file in Notepad++. It defines:

- A weekly schedule (`cron: "0 1 * * 1"`, Mondays 01:00 UTC) against the `master` branch, with manual triggering (`trigger: none`).
- Two boolean parameters: `publishReports` and `isSendMail`.
- A resource reference to `RepoWikiMonitoring` (`RxR-Architecture/rxi-monitoring-wiki`).
- Variable groups `RxI-Monitoring-Naples` and `CIsharedvariables`.
- A single stage (`StageMonitoringPhysicalPartitionOverSize`) and job (`RunningMonitoring`) that:
  1. Cleans the artifacts folder.
  2. Downloads the previous report from Blob Storage.
  3. Runs `monitor-ru.ps1` to generate the RU report against Cosmos DB.
  4. Publishes the report as a build artifact.
  5. Publishes the report to Blob Storage.
  6. Updates the monitoring wiki via a `git` push.

## 2. Corrections after re-shared (higher-resolution) screenshots

- Fixed the **argument order** for the "Download From Blob Storage" step. It was `-DestinationPath ... -BlobName ...`; corrected to match the source: `-AccountName -Container -ResourceGroup -BlobName -DestinationPath`.
- Preserved the source's **multi-line double-quoted YAML string** for that same `arguments` field (valid YAML — line breaks fold to spaces on parse), matching the original file's exact line count.

## 3. Linting

Validated the file with:

- `pyyaml` (`yaml.safe_load`) — confirms the file parses without syntax errors.
- A duplicate-key scan — no duplicate keys found.
- A tabs/trailing-whitespace scan — none found.
- `yamllint` (strict style mode) — flagged only cosmetic items (missing space after `#` in two comments, and non-strict-standard indentation on the `${{ if }}/${{ else }}` template blocks). These match the original source formatting and are not functional issues; the file is valid Azure Pipelines YAML.

## 4. Added non-prod support

A second pipeline (a ServiceBus DLQ/queue-depth monitoring pipeline) was used as a reference for how non-prod environments are structured, and that pattern was applied to `monitor-ru.yml`. All changes below are additive — no production values were altered.

### 4a. Environment-specific account/subscription variables (placeholders)

Previously, `accountName`, `resourceGroup`, `subscriptionName`, `subscriptionId`, and `subscriptionCosmosId` were flat, prod-only values. They are now wrapped in an `${{ if eq(variables['envName'], 'production') }} / ${{ else }}` block, matching the pattern already used for the repo/folder-path variables:

| Variable | Prod value (unchanged) | Non-prod value (placeholder) |
|---|---|---|
| `accountName` | `rxr-rxi-prod-cus-cosmos-01` | `<NONPROD_COSMOS_ACCOUNT_NAME>` |
| `resourceGroup` | `rxr-rxi-prod-cus-rg` | `<NONPROD_RESOURCE_GROUP_NAME>` |
| `subscriptionName` | `rpu-prod-rxrenewal-01` | `<NONPROD_SUBSCRIPTION_NAME>` |
| `subscriptionId` | `fa3f7777-9616-4674-8e74-dcf34fde8aa6` | `<NONPROD_SUBSCRIPTION_ID>` |
| `subscriptionCosmosId` | `ed7d3def-6e67-4f63-9d34-fdadcd09ab12` | `<NONPROD_SUBSCRIPTION_COSMOS_ID>` |

### 4b. `envName` is now selectable, not hardcoded

- Added a new pipeline parameter:
  ```yaml
  - name: envName
    displayName: "Environment"
    type: string
    default: "production"
    values:
      - production
      - nonprod
  ```
- The `envName` variable now reads `${{ parameters.envName }}` instead of the fixed string `"production"`, so it actually drives every `if/else` branch in the file (previously those branches always resolved to the `production` case).

### 4c. Agent pool now environment-aware (placeholder)

```yaml
pool:
  ${{ if eq(parameters.envName, 'production') }}:
    name: rxi-be17-agent-pool-prod
  ${{ else }}:
    name: "<NONPROD_AGENT_POOL_NAME>"
```

### 4d. "Generate Reports" service connection now environment-aware (placeholder)

```yaml
inputs:
  ${{ if eq(variables['envName'], 'production') }}:
    azureSubscription: "RxI-Prod-Leap"
  ${{ else }}:
    azureSubscription: "<NONPROD_SERVICE_CONNECTION_NAME>"
  scriptType: "pscore"
  ...
```

## 5. Full inventory: what is / isn't environment-aware

**Now environment-aware (prod + non-prod branches):**
- `envName` (parameter-driven)
- `pool` name
- `repositoryMonitoringPath`, `repositoryMonitoringUtilsPath`, `reportsFolder`, `wikiBuildFolder`, `wikiADOFolder`
- `accountName`, `resourceGroup`, `subscriptionName`, `subscriptionId`, `subscriptionCosmosId`
- `azureSubscription` (Generate Reports task)

**Still a single, fixed value regardless of environment** (by design — shared infra, same in the reference pipeline for its own non-prod build):
- `serviceConnectionNameStorage` (`RxI-PERF-05-BLOB`)
- `StorageAccountName` (`rxrrxiprf01cusst`)
- `ContainerName` (`rxi-monitoring-reports`)

## 6. Placeholders that must be replaced before running against non-prod

| Placeholder | Location |
|---|---|
| `<NONPROD_AGENT_POOL_NAME>` | `pool.name` (else branch) |
| `<NONPROD_SERVICE_CONNECTION_NAME>` | "Generate Reports" task `azureSubscription` (else branch) |
| `<NONPROD_COSMOS_ACCOUNT_NAME>` | stage `variables.accountName` (else branch) |
| `<NONPROD_RESOURCE_GROUP_NAME>` | stage `variables.resourceGroup` (else branch) |
| `<NONPROD_SUBSCRIPTION_NAME>` | stage `variables.subscriptionName` (else branch) |
| `<NONPROD_SUBSCRIPTION_ID>` | stage `variables.subscriptionId` (else branch) |
| `<NONPROD_SUBSCRIPTION_COSMOS_ID>` | stage `variables.subscriptionCosmosId` (else branch) |

Search the file for `NONPROD` to find every placeholder needing a real value.
