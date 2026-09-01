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
          timeoutInMinutes: 60   # must be > the task timeout below (MS docs); task timeout is the real limit
          steps:
            - task: ManualValidation@0
              # DEVIATION from helm reference: helm leaves this env-level gate's
              # timeout unset (uses the shared template's default). We use an
              # inline ManualValidation task with an explicit, short timeout
              # instead - kept deliberately from earlier testing of the
              # auto-reject behavior, not matched to helm.
              timeoutInMinutes: 2   # gate auto-rejects after 2 minutes
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
          # DEVIATION from helm reference: helm leaves this unset (uses the shared
          # template's default). Set explicitly here so Break-Glass doesn't wait
          # indefinitely; kept from earlier testing rather than left unset.
          timeoutInMinutes: 10
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


