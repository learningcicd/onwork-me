{% raw %}
```yaml
trigger: none

appendCommitMessageToRunName: false

parameters:
  - name: branchName
    displayName: Branch Name
    type: string
    default: 'heads/develop'
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
    value: "dev,e2e,e2e-2,perf,perf-2"
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
        - ${{ if eq(env, 'dev') }}:
          - name: deployServiceConnection
            value: RxI-Dev2-Leap-05
          - name: functionAppName
            value: rxr-rxi-dev-01-cus-fa-purchaseordermanagement
        - ${{ if eq(env, 'e2e') }}:
          - name: deployServiceConnection
            value: RxI-E2E-Leap-05
          - name: functionAppName
            value: rxr-rxi-e2e-01-cus-fa-purchaseordermanagement
        - ${{ if eq(env, 'e2e-2') }}:
          - name: deployServiceConnection
            value: RxI-E2E-02-Leap
          - name: functionAppName
            value: rxr-rxi-e2e-02-cus-fa-purchaseordermanagement
        - ${{ if eq(env, 'perf') }}:
          - name: deployServiceConnection
            value: RxI-PERF-05
          - name: functionAppName
            value: rxr-rxi-perf-01-cus-fa-purchaseordermanagement
        - ${{ if eq(env, 'perf-2') }}:
          - name: deployServiceConnection
            value: RxI-PERF-05
          - name: functionAppName
            value: rxr-rxi-perf-02-cus-fa-purchaseordermanagement
        - name: appSettingsFile
          value: '$(Build.SourcesDirectory)/temp/$(Build.BuildId)/${{ env }}/app-settings.json'
      jobs:
        - template: jobs/job-manual-approval.yml@rxiPipelineTemplate
          parameters:
            jobName: approvalJob
            timeoutInMinutes: 10
            # NOTE! Possible defect! I'm unable to use an email in this list, although Microsoft docs state that I should be able to!
            approvers: |
              ${{ variables.testingStageApprovers }}
        - job: deployJob
          dependsOn: approvalJob
          displayName: Deploy ${{ env }}
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
                    # strip any non-numeric suffix (e.g. a pasted .zip extension) from the build id
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
                  branchName: 'refs/${{ parameters.branchName }}'
                  pipelineId: '$(pipelineBuildId)'
                  artifactName: '$(artifactName)'
                  itemPattern: '**'
                  targetPath: '$(downloadedArtifacts)/${{ env }}'
            - pwsh: |
                Write-Host "Artifact download complete for ${{ env }}"
              displayName: "Confirm artifact download"
            - task: PowerShell@2
              name: checkAzureEnvironment
              displayName: Check Azure Environment
              inputs:
                targetType: 'inline'
                script: |
                  # Get-InstalledModule -Name Az.Accounts
                  $azVersion = az --version | ForEach-Object {
                    $_ -match '\s*azure-cli\s+(?<version>\d+\.\d+\.\d+)\s*'
                    $Matches['version']
                  } | Where-Object { !!$_ } | Select-Object -First 1

                  Write-Host "Az CLI version: $($azVersion)"
                failOnStderr: false
                showWarnings: true
                pwsh: true
            - task: PowerShell@2
              name: buildAppSettings
              displayName: Build App Settings
              condition: eq(${{ parameters.deployAppSettings }}, true)
              inputs:
                targetType: 'inline'
                script: |
                  New-Item -Path '$(appSettingsFile)' -Value '' -Type File -Force
                  & "$(Build.SourcesDirectory)/scripts/build-app-settings.ps1" -EnvName '${{ env }}' -OutputPath '$(appSettingsFile)'
                failOnStderr: true
                showWarnings: true
                pwsh: true
            - template: dotnet/functions/deploy/deploy-common-template.yml@pipelineTemplate
              parameters:
                appSettingsPath: '$(appSettingsFile)'
                appName: '$(functionAppName)'
                package: '$(downloadedArtifacts)/${{ env }}/**/*${{ parameters.artifactName }}*.zip'
                healthcheckEnabled: false
                functionAppDeployEnabled: ${{ parameters.deployCode }}
                functionAppSettingsPublishEnabled: ${{ parameters.deployAppSettings }}
                # service connection MUST be a literal resolved at compile time -
                # a $( ) runtime macro cannot be authorized by Azure DevOps
                ${{ if eq(env, 'dev') }}:
                  serviceConnection: RxI-Dev2-Leap-05
                ${{ if eq(env, 'e2e') }}:
                  serviceConnection: RxI-E2E-Leap-05
                ${{ if eq(env, 'e2e-2') }}:
                  serviceConnection: RxI-E2E-02-Leap
                ${{ if eq(env, 'perf') }}:
                  serviceConnection: RxI-PERF-05
                ${{ if eq(env, 'perf-2') }}:
                  serviceConnection: RxI-PERF-05
################################################ END NONPROD DEPLOYMENTS ################################################

  - stage: BreakGlassApproval
    displayName: Break-Glass Approval
    dependsOn: []
    isSkippable: false
    jobs:
      - template: jobs/job-manual-approval.yml@rxiPipelineTemplate
        parameters:
          jobName: approvalJob
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

################################################ START QE APPROVAL STAGE ################################################
  - stage: QEFinalApproval
    displayName: QE Review & Approval for Prod
    dependsOn:
      - deploy_stage_dev
      - deploy_stage_e2e
      - deploy_stage_e2e_2
      - deploy_stage_perf
      - deploy_stage_perf_2
      - BreakGlassApproval
    condition: |
      or(
        succeeded('deploy_stage_dev'),
        succeeded('deploy_stage_e2e'),
        succeeded('deploy_stage_e2e_2'),
        succeeded('deploy_stage_perf'),
        succeeded('deploy_stage_perf_2'),
        succeeded('BreakGlassApproval')
      )
    isSkippable: false
    jobs:
      - template: jobs/job-manual-approval.yml@rxiPipelineTemplate
        parameters:
          jobName: qe_final_approval
          displayName: QE Review & Approval
          timeoutInMinutes: 10
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
      - name: deployServiceConnection
        value: "TODO-RxI-PROD-service-connection"
      - name: functionAppName
        value: "TODO-rxr-rxi-prod-01-cus-fa-purchaseordermanagement"
      - name: appSettingsFile
        value: '$(Build.SourcesDirectory)/temp/$(Build.BuildId)/prod/app-settings.json'
    jobs:
      - template: jobs/job-manual-approval.yml@rxiPipelineTemplate
        parameters:
          jobName: prod_approval
          displayName: Approve Prod Deploy
          timeoutInMinutes: 10
          approvers: |
            ${{ variables.deployApprovers }}
      - job: prod_deploy
        displayName: Run Prod Deployment
        dependsOn: prod_approval
        condition: succeeded('prod_approval')
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
                  # strip any non-numeric suffix (e.g. a pasted .zip extension) from the build id
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
                branchName: 'refs/${{ parameters.branchName }}'
                pipelineId: '$(pipelineBuildId)'
                artifactName: '$(artifactName)'
                itemPattern: '**'
                targetPath: '$(downloadedArtifacts)/prod'
          - pwsh: |
              Write-Host "Artifact download complete for prod (deployment steps are commented out)"
            displayName: "Confirm artifact download"
          # - task: PowerShell@2
          #   name: buildAppSettings
          #   displayName: Build App Settings
          #   condition: eq(${{ parameters.deployAppSettings }}, true)
          #   inputs:
          #     targetType: 'inline'
          #     script: |
          #       New-Item -Path '$(appSettingsFile)' -Value '' -Type File -Force
          #       & "$(Build.SourcesDirectory)/scripts/build-app-settings.ps1" -EnvName 'prod' -OutputPath '$(appSettingsFile)'
          #     failOnStderr: true
          #     showWarnings: true
          #     pwsh: true
          # NOTE: serviceConnection below is already a compile-time literal -
          #       do NOT change it back to '$(deployServiceConnection)', a runtime
          #       macro cannot be resolved or authorized by Azure DevOps.
          # - template: dotnet/functions/deploy/deploy-common-template.yml@pipelineTemplate
          #   parameters:
          #     appSettingsPath: '$(appSettingsFile)'
          #     appName: '$(functionAppName)'
          #     package: '$(downloadedArtifacts)/prod/**/*${{ parameters.artifactName }}*.zip'
          #     # TODO: replace with the real prod service connection name (literal, not a variable)
          #     serviceConnection: TODO-RxI-PROD-service-connection
          #     healthcheckEnabled: false
          #     functionAppDeployEnabled: ${{ parameters.deployCode }}
          #     functionAppSettingsPublishEnabled: ${{ parameters.deployAppSettings }}
################################################ END PROD DEPLOYMENT STAGE ################################################
################################################ START PRODFIX DEPLOYMENT STAGES ################################################
  - stage: post_deploy_stage_prodfix
    displayName: "Deploy to prodfix"
    dependsOn: DeployToProd
    condition: succeeded('DeployToProd')
    variables:
      - name: deployServiceConnection
        value: RxI-PRODFIX-05
      - name: functionAppName
        value: rxr-rxi-prodfix-01-cus-fa-purchaseordermanagement
      - name: appSettingsFile
        value: '$(Build.SourcesDirectory)/temp/$(Build.BuildId)/prodfix/app-settings.json'
    jobs:
      - template: jobs/job-manual-approval.yml@rxiPipelineTemplate
        parameters:
          jobName: prodfix_approval
          timeoutInMinutes: 10
          approvers: |
            ${{ variables.devopsApprovers }},
            ${{ variables.qeApprovers }}
      - job: prodfix_deploy
        displayName: "Post-Release Deploy to prodfix"
        dependsOn: prodfix_approval
        condition: succeeded('prodfix_approval')
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
                  # strip any non-numeric suffix (e.g. a pasted .zip extension) from the build id
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
                branchName: 'refs/${{ parameters.branchName }}'
                pipelineId: '$(pipelineBuildId)'
                artifactName: '$(artifactName)'
                itemPattern: '**'
                targetPath: '$(downloadedArtifacts)/prodfix'
          - pwsh: |
              Write-Host "Artifact download complete for prodfix (deployment steps are commented out)"
            displayName: "Confirm artifact download"
          # - task: PowerShell@2
          #   name: buildAppSettings
          #   displayName: Build App Settings
          #   condition: eq(${{ parameters.deployAppSettings }}, true)
          #   inputs:
          #     targetType: 'inline'
          #     script: |
          #       New-Item -Path '$(appSettingsFile)' -Value '' -Type File -Force
          #       & "$(Build.SourcesDirectory)/scripts/build-app-settings.ps1" -EnvName 'prodfix' -OutputPath '$(appSettingsFile)'
          #     failOnStderr: true
          #     showWarnings: true
          #     pwsh: true
          # NOTE: serviceConnection below is already a compile-time literal -
          #       do NOT change it back to '$(deployServiceConnection)', a runtime
          #       macro cannot be resolved or authorized by Azure DevOps.
          # - template: dotnet/functions/deploy/deploy-common-template.yml@pipelineTemplate
          #   parameters:
          #     appSettingsPath: '$(appSettingsFile)'
          #     appName: '$(functionAppName)'
          #     package: '$(downloadedArtifacts)/prodfix/**/*${{ parameters.artifactName }}*.zip'
          #     serviceConnection: RxI-PRODFIX-05
          #     healthcheckEnabled: false
          #     functionAppDeployEnabled: ${{ parameters.deployCode }}
          #     functionAppSettingsPublishEnabled: ${{ parameters.deployAppSettings }}
################################################ END PRODFIX DEPLOYMENT STAGES ################################################

```
{% endraw %}
