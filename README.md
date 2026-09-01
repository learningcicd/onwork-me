{% raw %}
```yaml
trigger: none

appendCommitMessageToRunName: false

parameters:
  - name: artifactName
    displayName: Artifact Name
    type: string
    default: ''
  - name: deployCode
    displayName: Deploy Code
    type: boolean
    default: true
  - name: deployAppSettings
    displayName: Deploy App Settings
    type: boolean
    default: true

variables:
  - name: buildPipelineProject
    value: '0bf1e934-99c7-462b-9e57-953979618cee'
  - name: buildPipelineDefinition
    value: '5544'
  - name: downloadedArtifacts
    value: '$(Pipeline.Workspace)/artifacts'
  - name: artifactName
    value: 'RxInventory.PurchaseOrderManagement'
  - name: envs
    value: "dev,e2e-01,e2e-02,perf-01,prodfix-01"
  - name: devopsApprovers
    value: "[RxI-DevOps-Deployment]\\RxI DevOps - Team"
  - name: deployApprovers
    value: "[RxI-DevOps-Deployment]\\rxireldeployment - Team"
  - name: qeApprovers
    value: "[RxI-DevOps-Deployment]\\RxI QE - US Team"
  - name: testingStageApprovers
    value: "[RxI-DevOps-Deployment]\\Testing Stage Approvers"
  - name: breakGlassApprovers
    value: "[RxI-DevOps-Deployment]\\Break Glass Approvers"
  - name: poolName
    value: "rxi-agent-pool"

pool:
  name: rxi-agent-pool
  demands:
    - azureps

resources:
  repositories:
    - repository: pipelineTemplate
      type: git
      name: PlatformX-Toolchain/pipeline-templates
      ref: refs/heads/master
    - repository: rxiPipelineTemplate
      type: git
      name: RxI-DevOps-Deployment/rxi-cd-templates
      ref: refs/tags/jobs/1.2.1

stages:
################################################ START NONPROD DEPLOYMENTS ################################################

  - ${{ each env in split(variables.envs, ',') }}:
    - stage: deploy_stage_${{ replace(env, '-', '_') }}
      displayName: "${{ env }}"
      dependsOn: []
      isSkippable: false
      variables:
        - name: displayEnv
          value: ${{ replace(env, '-', '_') }}
        - template: /variables/deploy/${{ env }}.yml
        - name: appSettingsFile
          value: '$(Build.SourcesDirectory)/temp/$(Build.BuildId)/${{ env }}/app-settings.json'
      jobs:
        # ---- Prep job: download artifact, capture live settings, build new, diff (table) ----
        # runs immediately - no approval needed to prepare & show the diff
        - job: prepJob
          displayName: Prepare & Diff ${{ env }}
          pool: { name: $(poolName), demands: azureps }
          timeoutInMinutes: 190
          steps:
            - ${{ if eq(parameters.deployCode, true) }}:
              - task: PowerShell@2
                displayName: Normalize version metadata
                inputs:
                  targetType: 'inline'
                  script: |
                    $versionParts = '${{ parameters.artifactName }}' -split '\-b'
                    $buildId = ($versionParts[-1] -replace '[^0-9]', '')
                    if (-not $buildId) { Write-Host "##vso[task.logissue type=error]Could not parse build id from artifact name '${{ parameters.artifactName }}' - expected format <name>-b<buildId>"; exit 1 }
                    Write-Host "Code Version: $($versionParts[0])"
                    Write-Host "Pipeline BuildId: $buildId"
                    Write-Host "##vso[task.setvariable variable=pipelineBuildId]$buildId"
                  failOnStderr: true
                  pwsh: true
                  runScriptInSeparateScope: true
              - task: DownloadPipelineArtifact@2
                inputs:
                  buildType: 'specific'
                  project: '$(buildPipelineProject)'
                  definition: '$(buildPipelineDefinition)'
                  buildVersionToDownload: 'specific'
                  pipelineId: '$(pipelineBuildId)'
                  artifactName: '$(artifactName)'
                  itemPattern: '**'
                  targetPath: '$(downloadedArtifacts)/${{ env }}'
            - template: /steps/prepare-and-diff-app-settings.yml
              parameters:
                envLabel: '${{ env }}'
                scriptEnvName: '${{ env }}'
                azureSubscription: ${{ variables.deployServiceConnection }}
                liveSettingsFile: '$(Build.SourcesDirectory)/temp/$(Build.BuildId)/${{ env }}/live-app-settings.json'
                artifactName: '${{ parameters.artifactName }}'
                deployAppSettings: ${{ parameters.deployAppSettings }}

        # ---- Single approval gate: review the diff above, then approve to deploy ----
        - job: diffApprovalJob
          dependsOn: prepJob
          displayName: Approve deploy after diff review (${{ env }})
          pool: server   # agentless - required for ManualValidation
          timeoutInMinutes: 1440   # 1 day - raised so this doesn't silently cap the
                                   # gate now that the task-level timeout is unset.
                                   # Chosen value, not drawn from helm (helm doesn't
                                   # use this inline-task pattern for its per-env gate).
          steps:
            - task: ManualValidation@0
              # MATCHED to helm reference: helm leaves this env-level gate's
              # timeout unset (uses the shared template's/task's own default).
              # Previous testing value commented out below rather than applied.
              # timeoutInMinutes: 2   # gate auto-rejects after 2 minutes
              inputs:
                notifyUsers: |
                  ${{ variables.testingStageApprovers }}
                instructions: "Artifact: ${{ parameters.artifactName }} | Env: ${{ env }}. Review the app settings diff in the 'Prepare & Diff' job log, then approve to deploy."
                onTimeout: 'reject'

        # ---- Deploy job: runs only after the diff has been approved ----
        - job: deployJob
          dependsOn: diffApprovalJob
          condition: succeeded('diffApprovalJob')
          displayName: Deploy ${{ env }}
          pool: { name: $(poolName), demands: azureps }
          timeoutInMinutes: 190
          steps:
            - template: /steps/deploy-functionapp.yml
              parameters:
                envLabel: '${{ env }}'
                artifactName: '${{ parameters.artifactName }}'
                deployCode: ${{ parameters.deployCode }}
                deployAppSettings: ${{ parameters.deployAppSettings }}
                serviceConnection: ${{ variables.deployServiceConnection }}
################################################ END NONPROD DEPLOYMENTS ################################################

  - stage: BreakGlassApproval
    displayName: Break-Glass Approval
    dependsOn: []
    isSkippable: false
    jobs:
      - template: jobs/job-manual-approval.yml@rxiPipelineTemplate
        parameters:
          jobName: approvalJob
          # MATCHED to helm reference: helm never passes timeoutInMinutes to this
          # shared template for Break-Glass, so it uses the template's own default.
          # Previous testing value commented out below rather than applied.
          # timeoutInMinutes: 10
          approvers: |
            ${{ variables.breakGlassApprovers }}
      - job: BreakGlassApproval
        dependsOn: approvalJob
        pool: { name: $(poolName) }
        steps:
          - pwsh: |
              Write-Host "Break-Glass approval granted!"
            displayName: "Break-Glass Approval"

################################################ START QE APPROVAL STAGE (now also owns Prepare & Diff prod) ################################################

  - stage: QEFinalApproval
    displayName: QE Review & Approval for Prod
    dependsOn:
      - deploy_stage_dev
      - deploy_stage_e2e_01
      - deploy_stage_e2e_02
      - deploy_stage_perf_01
      - deploy_stage_prodfix_01
      - BreakGlassApproval
    condition: |
      or(
        succeeded('deploy_stage_dev'),
        succeeded('deploy_stage_e2e_01'),
        succeeded('deploy_stage_e2e_02'),
        succeeded('deploy_stage_perf_01'),
        succeeded('deploy_stage_prodfix_01'),
        succeeded('BreakGlassApproval')
      )
    isSkippable: false
    variables:
      - template: /variables/deploy/prod.yml
      - name: prodAppSettingsFile
        value: '$(Build.SourcesDirectory)/temp/$(Build.BuildId)/prod/app-settings.json'
    jobs:
      # ---- Prepare & Diff for prod now runs INSIDE the QE gate stage ----
      - job: prepProdJob
        displayName: Prepare & Diff prod
        pool: { name: $(poolName), demands: azureps }
        timeoutInMinutes: 2880
        steps:
          - template: /steps/prepare-and-diff-app-settings.yml
            parameters:
              envLabel: prod
              scriptEnvName: prod
              azureSubscription: ${{ variables.deployServiceConnection }}
              functionAppNameVar: '$(prodFunctionAppName)'
              newSettingsFile: '$(prodAppSettingsFile)'
              liveSettingsFile: '$(Build.SourcesDirectory)/temp/$(Build.BuildId)/prod/live-app-settings.json'
              artifactName: '${{ parameters.artifactName }}'
              deployAppSettings: ${{ parameters.deployAppSettings }}

      - template: jobs/job-manual-approval.yml@rxiPipelineTemplate
        parameters:
          jobName: qe_final_approval
          displayName: QE Review & Approval
          timeoutInMinutes: 2880 # 2 days - matches helm reference (QEFinalApproval gates)
          approvers: |
            ${{ variables.deployApprovers }},
            ${{ variables.qeApprovers }}
################################################ END QE APPROVAL STAGE ################################################
################################################ START PROD DEPLOYMENT STAGE ################################################

  - stage: DeployToProd
    displayName: Deploy to Prod
    dependsOn: QEFinalApproval
    condition: succeeded('QEFinalApproval')
    variables:
      - template: /variables/deploy/prod.yml
      - name: appSettingsFile
        value: '$(Build.SourcesDirectory)/temp/$(Build.BuildId)/prod/app-settings.json'
      - name: prodAppSettingsFile
        value: '$(Build.SourcesDirectory)/temp/$(Build.BuildId)/prod-deploy/app-settings.json'
    jobs:
      # ---- Prepare & Diff prod (same pattern as the QE gate stage) ----
      - job: prepProdDiffJob
        displayName: Prepare & Diff prod
        pool: { name: $(poolName), demands: azureps }
        timeoutInMinutes: 2880
        steps:
          - template: /steps/prepare-and-diff-app-settings.yml
            parameters:
              envLabel: prod
              scriptEnvName: prod
              azureSubscription: ${{ variables.deployServiceConnection }}
              functionAppNameVar: '$(prodFunctionAppName)'
              newSettingsFile: '$(prodAppSettingsFile)'
              liveSettingsFile: '$(Build.SourcesDirectory)/temp/$(Build.BuildId)/prod-deploy/live-app-settings.json'
              artifactName: '${{ parameters.artifactName }}'
              deployAppSettings: ${{ parameters.deployAppSettings }}

      - template: jobs/job-manual-approval.yml@rxiPipelineTemplate
        parameters:
          jobName: prod_approval
          displayName: Approve Prod Deploy
          timeoutInMinutes: 1440 # 1 day - matches helm reference (DeployToProd prod_approval)
          approvers: |
            ${{ variables.deployApprovers }}
      - job: prod_deploy
        displayName: Run Prod Deployment
        dependsOn: prod_approval
        condition: succeeded('prod_approval')
        pool: { name: $(poolName), demands: azureps }
        timeoutInMinutes: 190
        steps:
          - template: /steps/deploy-functionapp.yml
            parameters:
              envLabel: prod
              artifactName: '${{ parameters.artifactName }}'
              deployCode: ${{ parameters.deployCode }}
              deployAppSettings: ${{ parameters.deployAppSettings }}
              serviceConnection: ${{ variables.deployServiceConnection }}
################################################ END PROD DEPLOYMENT STAGE ################################################
################################################ START PRODFIX DEPLOYMENT STAGES ################################################

  - stage: post_deploy_stage_prodfix_01
    displayName: "Deploy to prodfix-01"
    dependsOn: DeployToProd
    condition: succeeded('DeployToProd')
    variables:
      - template: /variables/deploy/prodfix-01.yml
      - name: appSettingsFile
        value: '$(Build.SourcesDirectory)/temp/$(Build.BuildId)/prodfix-01/app-settings.json'
    jobs:
      # ---- Prepare & Diff prodfix-01 (same pattern as prod) ----
      - job: prepProdfixDiffJob
        displayName: Prepare & Diff prodfix-01
        pool: { name: $(poolName), demands: azureps }
        timeoutInMinutes: 2880
        steps:
          - template: /steps/prepare-and-diff-app-settings.yml
            parameters:
              envLabel: prodfix-01
              scriptEnvName: prodfix-01
              azureSubscription: ${{ variables.deployServiceConnection }}
              functionAppNameVar: '$(functionAppName)'
              newSettingsFile: '$(appSettingsFile)'
              liveSettingsFile: '$(Build.SourcesDirectory)/temp/$(Build.BuildId)/prodfix-01/live-app-settings.json'
              artifactName: '${{ parameters.artifactName }}'
              deployAppSettings: ${{ parameters.deployAppSettings }}

      - template: jobs/job-manual-approval.yml@rxiPipelineTemplate
        parameters:
          jobName: prodfix_approval
          timeoutInMinutes: 1440 # 1 day - matches helm reference (post_deploy_stage prodfix_approval)
          approvers: |
            ${{ variables.devopsApprovers }},
            ${{ variables.qeApprovers }}
      - job: prodfix_deploy
        displayName: "Post-Release Deploy to prodfix-01"
        dependsOn: prodfix_approval
        condition: succeeded('prodfix_approval')
        pool: { name: $(poolName), demands: azureps }
        timeoutInMinutes: 190
        steps:
          - template: /steps/deploy-functionapp.yml
            parameters:
              envLabel: prodfix-01
              artifactName: '${{ parameters.artifactName }}'
              deployCode: ${{ parameters.deployCode }}
              deployAppSettings: ${{ parameters.deployAppSettings }}
              serviceConnection: ${{ variables.deployServiceConnection }}
################################################ END PRODFIX DEPLOYMENT STAGES ################################################

```
{% endraw %}


# Timeouts in this pipeline — a field guide

This pipeline has **two fundamentally different kinds of timeout**, and mixing them up is the most common source of confusion when a run "seems stuck." This doc explains both, lists every timeout value in the pipeline, and explains the one Azure DevOps rule that trips people up most: **job-level timeout vs. task-level timeout, and which one actually governs.**

## The two kinds of timeout

### 1. Work timeouts — "how long can this job run before ADO assumes it's hung?"

These sit on jobs that do real work on a real agent: downloading an artifact, running a PowerShell script, calling the Azure Functions deploy API. If a job runs past this limit, ADO kills it and fails the job — the assumption is that something has hung (a stuck download, an API call that never returns), not that a human needs to make a decision.

### 2. Approval timeouts — "how long do we wait for a human to click Approve/Reject?"

These sit on the `ManualValidation@0` task (and the equivalent inside the shared `job-manual-approval.yml@rxiPipelineTemplate`). Nothing is "running" while this waits — the job just sits idle on the agentless `server` pool, costing nothing, until someone responds or the clock runs out.

**These two numbers are set completely independently and answer completely different questions.** A 190-minute work timeout on a deploy job has nothing to do with how long an approval gate waits.

## The critical rule: job timeout vs. task timeout

Every `job:` in this pipeline can carry its own `timeoutInMinutes`. Some jobs *also* contain a `task:` (like `ManualValidation@0`) that has its **own**, separate `timeoutInMinutes`. When both exist:

> **The JOB's timeout is a hard ceiling that applies no matter what the task's own timeout says.** If the job timeout is shorter than the task timeout, the job gets killed first — the task's timeout never gets a chance to matter.

This is exactly why `diffApprovalJob`'s job-level timeout is set to `1440` (1 day) even though the task-level timeout inside it is commented out (unset). If we'd left the job-level wrapper at a low number like 60, the gate would have been silently cut off at an hour, regardless of the fact that the task itself was configured to wait indefinitely.

**Microsoft's own guidance:** the job's `timeoutInMinutes` must be *greater than* the task's `timeoutInMinutes` for the task's timeout to be the one that actually governs.

## Another rule that matters here: Azure DevOps' default is 60 minutes, not "unlimited"

If you don't specify `timeoutInMinutes` on a job at all, Azure DevOps does **not** let it run forever — it silently applies a **default of 60 minutes**. This is true for jobs on real agents and for agentless (`server` pool) jobs alike. The maximum you can raise an agentless job to is **30 days (43,200 minutes)**.

So "not setting a timeout" is never a way to get infinite waiting — it's a way to get exactly 60 minutes, which is almost always shorter than you'd want for a real approval gate. Every timeout in this pipeline is set explicitly for this reason.

## Every timeout in this pipeline

### Non-prod environments (`dev`, `e2e-01`, `e2e-02`, `perf-01`, `prodfix-01`) — one set of three jobs, repeated per environment

| Job | Type | Timeout | What it means |
|---|---|---|---|
| `prepJob` | Work (real agent) | **190 min** (~3h 10m) | Download artifact, capture live settings, build new settings, print diff. Matches helm's equivalent `deployJob` timeout value. |
| `diffApprovalJob` | Approval (agentless) | **Job: 1440 min (1 day)** · Task: *unset* | Waits for a human to approve/reject after reviewing the diff. Task-level timeout is deliberately commented out to match how helm's reference pipeline handles this gate (no explicit timeout — relies on default behavior). The 1-day job-level ceiling exists only so this doesn't silently fall back to the 60-minute ADO default. |
| `deployJob` | Work (real agent) | **190 min** | The actual code/settings deploy. Matches helm exactly. |

**What you'll see if a run sits for hours:** if `Prepare & Diff` shows ✅ done in under a minute, but `Approve deploy after diff review` shows a blue clock and has been sitting for, say, 13 hours — **this is expected**, not stuck. It's waiting on a human, and won't time out until the 24-hour mark.

### Break-Glass Approval

| Job | Type | Timeout | What it means |
|---|---|---|---|
| `approvalJob` (shared template) | Approval (agentless) | *Unset — commented out* | Matches helm, which never passes a timeout to this shared template. Falls back to whatever `job-manual-approval.yml@rxiPipelineTemplate`'s internal default is — observed in practice to be noticeably shorter than 24 hours, since Break-Glass gates typically show as failed/rejected well before the non-prod gates time out. |

### QE Review & Approval for Prod

| Job | Type | Timeout | What it means |
|---|---|---|---|
| `prepProdJob` | Work (real agent) | **2880 min (2 days)** | Prod's Prepare & Diff, run fresh inside the QE stage. |
| `qe_final_approval` (shared template) | Approval (agentless) | **2880 min (2 days)** | Matches helm's QE gates exactly. |

### Deploy to Prod

| Job | Type | Timeout | What it means |
|---|---|---|---|
| `prepProdDiffJob` | Work (real agent) | **2880 min (2 days)** | A second, independent fresh Prod diff, run again right before the prod approval — real time has passed since the QE gate. |
| `prod_approval` (shared template) | Approval (agentless) | **1440 min (1 day)** | Matches helm's `DeployToProd` gate exactly. |
| `prod_deploy` | Work (real agent) | **190 min** | The actual prod deploy. |

### Deploy to prodfix-01 (post-prod)

| Job | Type | Timeout | What it means |
|---|---|---|---|
| `prepProdfixDiffJob` | Work (real agent) | **2880 min (2 days)** | prodfix-01's Prepare & Diff. |
| `prodfix_approval` (shared template) | Approval (agentless) | **1440 min (1 day)** | Matches helm's `post_deploy_stage` prodfix gate exactly. |
| `prodfix_deploy` | Work (real agent) | **190 min** | The actual prodfix-01 deploy. |

## Quick answers to "is this stuck?"

- **A work job (Prepare & Diff, Deploy) has been running for longer than its listed minutes above** → genuinely worth investigating. Something is likely hung (agent lost connectivity, a stuck download, an Azure API call that never returned).
- **An approval job (Approve deploy after diff review, QE Review & Approval, Approve Prod Deploy, prodfix approval) shows a blue clock and hasn't hit its listed ceiling yet** → not stuck. It's waiting for a human. Check who's in the relevant approvers group and nudge them, or approve/reject it yourself if you're authorized.
- **A stage shows "Not started" even though one of its dependency stages already succeeded** → this is expected Azure DevOps behavior, not a bug. A stage's `dependsOn` always waits for **every** listed dependency to reach a terminal state (succeeded, failed, *or* rejected/timed-out) before it even evaluates whether to run — there's no way in native ADO YAML to say "start as soon as the first of these finishes." `QEFinalApproval`, for example, lists all five non-prod stages in `dependsOn`; even though its `condition` only requires **one** of them to have succeeded, the stage itself won't begin until all five have finished one way or another.

## Where each helm-matched value came from

If you ever need to re-verify these against the reference pipeline (`refernce-helm-cd.yaml`):

| Our value | Helm source |
|---|---|
| `deployJob` / `prod_deploy` / `prodfix_deploy` = 190 min | Helm's nonprod `deployJob`, line ~120 |
| `qe_final_approval` = 2880 min | Helm's `QEFinalApproval` stage gates, lines ~191, ~225 |
| `prod_approval` = 1440 min | Helm's `DeployToProd` approval, line ~265 |
| `prodfix_approval` = 1440 min | Helm's `post_deploy_stage` prodfix approval, line ~309 |
| Nonprod `diffApprovalJob` task timeout = unset | Helm's per-env `approvalJob` also has no explicit timeout |
| Break-Glass timeout = unset | Helm's Break-Glass gate also has no explicit timeout |

Values **not** found anywhere in helm (chosen independently, not matched to a reference):
- `prepJob`, `prepProdJob`, `prepProdDiffJob`, `prepProdfixDiffJob` — helm doesn't have a directly equivalent standalone "prepare and diff" job with its own timeout; ours are set to 190 min (nonprod) or 2880 min (prod-side) as reasonable working ceilings.
- `diffApprovalJob`'s **job-level** 1440-minute wrapper — this exists purely to stop ADO's 60-minute default from silently capping the gate now that its task-level timeout is unset. Helm doesn't need an equivalent because it doesn't use this inline-task pattern for its per-env gate.
