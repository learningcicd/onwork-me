{% raw %}
```yaml



```
{% endraw %}

# RxInventory.PurchaseOrderManagement — Function App CD Pipeline

This document explains what `reference-functionapp-cd.yaml` does, how it's structured, and how to run it. It reflects the current state of the pipeline after the recent restructuring (approval gates, app-settings diffing, environment renames, and service connection consolidation).

## What this pipeline does, at a glance

On every run, it:

1. Downloads a specific build artifact by build ID (parsed from the Artifact Name you provide).
2. For each non-prod environment, generates the *new* app settings, compares them against what's *currently live*, and prints a readable diff.
3. Gates each deployment behind a manual approval that **auto-rejects after 2 minutes** if nobody responds.
4. Deploys to non-prod environments in parallel.
5. Once at least one non-prod environment (or a Break-Glass override) succeeds, opens a QE approval gate — itself gated on a successful prod app-settings diff.
6. On QE approval, deploys to Prod, then to a post-prod `prodfix-01` environment.

## Environments

| Env id | Function App | Subscription |
|---|---|---|
| `dev` | `rxr-rxi-dev-01-cus-fa-purchaseordermanagement` | `RxI-NProd-05` |
| `e2e-01` | `rxr-rxi-e2e-01-cus-fa-purchaseordermanagement` | `RxI-NProd-05` |
| `e2e-02` | `rxr-rxi-e2e-02-cus-fa-purchaseordermanagement` | `RxI-NProd-05` |
| `perf-01` | `rxr-rxi-perf-01-cus-fa-purchaseordermanagement` | `RxI-NProd-05` |
| `prodfix-01` (non-prod validation pass) | `rxr-rxi-prodfix-01-cus-fa-purchaseordermanagement` | `RxI-NProd-05` |
| **Prod** | `rxr-rxi-prod-cus-fa-purchaseordermanagement` | `Rxi-prod05-static-ui-RxR-SCM` |
| **prodfix-01** (post-prod hotfix pass) | `rxr-rxi-prodfix-01-cus-fa-purchaseordermanagement` | `RxI-PRODFIX-05` |

All five non-prod environments share **one** subscription (`RxI-NProd-05`) — there's no per-environment subscription branching anymore. `prodfix-01` appears twice by design: once as an early non-prod validation pass, and again after Prod as a genuine post-prod hotfix stage with its own approval and its own (separate) service connection.

## Pipeline stages, in order

```
┌─────────────────────────────────────────────────────────────┐
│  dev · e2e-01 · e2e-02 · perf-01 · prodfix-01  (parallel)    │
│  each: Prepare & Diff → Approve (2 min) → Deploy              │
└─────────────────────────────────────────────────────────────┘
                    │ (at least one must succeed)
                    ▼
         Break-Glass Approval  (parallel escape hatch)
                    │
                    ▼
     QE Review & Approval for Prod
     (requires the prod diff to succeed AND
      a non-prod env / Break-Glass to succeed)
                    │
                    ▼
              Deploy to Prod
     Prepare & Diff prod → Approve → Deploy
                    │
                    ▼
           Deploy to prodfix-01
     Prepare & Diff prodfix-01 → Approve → Deploy
```

### Non-prod environments

Each of `dev`, `e2e-01`, `e2e-02`, `perf-01`, `prodfix-01` runs independently and in parallel (no dependency between them). Within each:

1. **Prepare & Diff** — downloads the artifact, captures the *live* app settings from the running function app, generates the *new* app settings from the repo's build script, and prints a diff.
2. **Approve deploy after diff review** — a manual approval gate. **Times out and auto-rejects after 2 minutes** if not actioned. Review the diff in the "Prepare & Diff" job log before approving.
3. **Deploy** — pushes code and/or app settings, depending on the run's `Deploy Code` / `Deploy App Settings` inputs.

### Break-Glass Approval

Runs in parallel with everything else. If approved, it satisfies the QE stage's entry condition on its own — a way to reach Prod without any non-prod environment having succeeded, for genuine emergencies. It has no functional effect other than being a recorded approval.

### QE Review & Approval for Prod

This stage only starts once **both**:
- the Prod app-settings diff has succeeded, **and**
- at least one non-prod environment (or Break-Glass) has succeeded.

It shows you the Prod diff and asks for approval before Prod deployment proceeds.

### Deploy to Prod

Runs its own **Prepare & Diff prod** step (build + capture + diff) immediately before the prod approval gate, so the diff you see is fresh at the point of approval — not the one shown earlier in QE. Approve, then it deploys.

### Deploy to prodfix-01 (post-prod)

Runs only after Prod succeeds. Same pattern: **Prepare & Diff prodfix-01** → approve → deploy. Uses its own service connection (`RxI-PRODFIX-05`), separate from the earlier non-prod `prodfix-01` pass.

## Understanding the app settings diff

Every "Prepare & Diff" step prints a table like this:

```
================ CHANGED ================
Setting                          ExistingValue          DesiredValue
-------                          -------------          ------------
MitigationFunctionStartDate      01/26/2023 00:00:00    2023-01-26T00:00:00Z

================ ADDED ================
Setting     ExistingValue     DesiredValue
-------     -------------     ------------
NewFlag                       true

================ LIVE-ONLY ================
Setting
-------
SomeManuallySetKey

================ UNCHANGED ================
Setting
-------
FUNCTIONS_WORKER_RUNTIME_VERSION
```

- **CHANGED** — exists in both places, value differs. This *will* be overwritten on deploy. Values are shown.
- **ADDED** — new setting, doesn't exist live yet. This *will* be created. Values are shown.
- **LIVE-ONLY** — exists live but isn't managed by this deploy. **The deploy merges, it does not delete** — these are left untouched. Values are not shown (they may be secrets).
- **UNCHANGED** — exists in both places with the same value already. Nothing happens. Values are not shown.

Date/time values that differ only in formatting (e.g. `01/26/2023 00:00:00` vs `2023-01-26T00:00:00Z`) are treated as the same value and classified UNCHANGED, not CHANGED — Azure normalizes the format on deploy regardless.

## How to run it

1. Open the pipeline in Azure DevOps and select **Run pipeline**.
2. Fill in the three inputs:
   - **Artifact Name** — the build artifact to deploy, in the form `<name>-b<buildId>` (e.g. `RxInventory.PurchaseOrderManagement-1.0.0-b2068817`). The pipeline parses the trailing number after `-b` as the source build ID to download from. **Do not include a file extension** (no `.zip`).
   - **Deploy Code** — whether to deploy the function app code package.
   - **Deploy App Settings** — whether to build, diff, and publish app settings.
3. Run. Each non-prod environment starts immediately.
4. **Watch for "Permission needed"** on any stage — the first time a service connection is used by this pipeline, Azure DevOps requires a one-time authorization. Click **Permit** when prompted, or pre-authorize under *Project Settings → Service connections → [connection] → Security* to skip this in future runs. The connections you may need to authorize: `RxI-NProd-05`, `Rxi-prod05-static-ui-RxR-SCM`, `RxI-PRODFIX-05`.
5. For each environment, review the "Prepare & Diff" job log, then approve or reject within **2 minutes** — the gate does not wait indefinitely.
6. Once a non-prod environment succeeds (or Break-Glass is approved), review and approve the QE gate, then the Prod gate, then the prodfix-01 gate.

## Known open items

- **Deployment steps for Prod and prodfix-01 (post-prod) are currently commented out** in the YAML — those stages will download artifacts and run their diffs, but the actual `az` deploy call is disabled pending sign-off. Nonprod environments deploy live today.
- **`branchName` was removed** as a pipeline parameter — the artifact is always resolved by exact build ID, so a branch filter was unused and has been dropped from the UI.
- The 2-minute approval timeout is set low for testing the auto-reject behavior. Before using this pipeline for real releases, increase `timeoutInMinutes` on the `ManualValidation@0` tasks to a realistic review window.
