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
        - ${{ if eq(env, 'dev') }}:
          - name: deployServiceConnection
            value: RxI-Dev2-Leap-05
          - name: functionAppName
            value: rxr-rxi-dev-01-cus-fa-purchaseordermanagement
        - ${{ if eq(env, 'e2e-01') }}:
          - name: deployServiceConnection
            value: RxI-NProd-05
          - name: functionAppName
            value: rxr-rxi-e2e-01-cus-fa-purchaseordermanagement
        - ${{ if eq(env, 'e2e-02') }}:
          - name: deployServiceConnection
            value: RxI-E2E-02-Leap
          - name: functionAppName
            value: rxr-rxi-e2e-02-cus-fa-purchaseordermanagement
        - ${{ if eq(env, 'perf-01') }}:
          - name: deployServiceConnection
            value: RxI-NProd-05
          - name: functionAppName
            value: rxr-rxi-perf-01-cus-fa-purchaseordermanagement
        - ${{ if eq(env, 'prodfix-01') }}:
          - name: deployServiceConnection
            value: RxI-PRODFIX-05
          - name: functionAppName
            value: rxr-rxi-prodfix-01-cus-fa-purchaseordermanagement
        - name: appSettingsFile
          value: '$(Build.SourcesDirectory)/temp/$(Build.BuildId)/${{ env }}/app-settings.json'
      jobs:
        # ---- Prep job: download artifact, capture live settings, build new, diff (table) ----
        # runs immediately - no approval needed to prepare & show the diff
        - job: prepJob
          displayName: Prepare & Diff ${{ env }}
          pool: { name: $(poolName), demands: azureps }
          timeoutInMinutes: 60
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
            # STEP 1: capture live (pre-deploy) settings; write to file for the diff
            - task: AzureCLI@2
              name: captureAppSettings
              displayName: "Capture existing app settings (${{ env }})"
              inputs:
                ${{ if eq(env, 'dev') }}:
                  azureSubscription: RxI-Dev2-Leap-05
                ${{ if eq(env, 'e2e-01') }}:
                  azureSubscription: RxI-NProd-05
                ${{ if eq(env, 'e2e-02') }}:
                  azureSubscription: RxI-E2E-02-Leap
                ${{ if eq(env, 'perf-01') }}:
                  azureSubscription: RxI-NProd-05
                ${{ if eq(env, 'prodfix-01') }}:
                  azureSubscription: RxI-PRODFIX-05
                scriptType: pscore
                scriptLocation: inlineScript
                inlineScript: |
                  $appName = "$(functionAppName)"
                  $liveFile = "$(Build.SourcesDirectory)/temp/$(Build.BuildId)/${{ env }}/live-app-settings.json"
                  New-Item -Path $liveFile -ItemType File -Force | Out-Null
                  $rg = az functionapp list --query "[?name=='$appName'].resourceGroup | [0]" -o tsv
                  if (-not $rg) {
                    Write-Host "##vso[task.logissue type=warning]Could not locate function app '$appName' in this subscription; skipping snapshot."
                    '[]' | Set-Content -Path $liveFile
                  } else {
                    Write-Host "Resource group: $rg"
                    az functionapp config appsettings list --name $appName --resource-group $rg --output json | Set-Content -Path $liveFile
                  }
            # STEP 2: build the new app settings file we intend to deploy
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
            # STEP 3: diff live vs new, rendered as an aligned TABLE in the log
            - task: PowerShell@2
              name: diffAppSettings
              displayName: "Diff app settings: live vs new (${{ env }})"
              condition: eq(${{ parameters.deployAppSettings }}, true)
              inputs:
                targetType: 'inline'
                pwsh: true
                script: |
                  $liveFile = "$(Build.SourcesDirectory)/temp/$(Build.BuildId)/${{ env }}/live-app-settings.json"
                  $newFile  = "$(appSettingsFile)"

                  function Read-Settings($path) {
                    $map = @{}
                    if (-not (Test-Path $path)) { return $map }
                    $raw = Get-Content -Path $path -Raw
                    if ([string]::IsNullOrWhiteSpace($raw)) { return $map }
                    try { $json = $raw | ConvertFrom-Json } catch { return $map }
                    foreach ($item in @($json)) {
                      if ($null -eq $item) { continue }
                      if ($item.PSObject.Properties.Name -contains 'name') { $map[$item.name] = "$($item.value)" }
                      else { foreach ($prop in $item.PSObject.Properties) { $map[$prop.Name] = "$($prop.Value)" } }
                    }
                    return $map
                  }

                  function Truncate($s, $n) { if ($null -eq $s) { return '' }; if ($s.Length -le $n) { return $s }; return $s.Substring(0, $n-3) + '...' }

                  # Treat two values as equal if their strings match OR they parse to the same date/time.
                  # This avoids false CHANGED rows when only the timestamp FORMAT differs
                  # (e.g. '08/28/2023 00:00:00' vs '2023-08-28T00:00:00.000Z') - Azure normalizes
                  # the format on deploy, so the value is effectively unchanged.
                  function ValuesEqual($a, $b) {
                    if ($a -eq $b) { return $true }
                    $da = [datetime]::MinValue; $db = [datetime]::MinValue
                    $okA = [datetime]::TryParse($a, [ref]$da)
                    $okB = [datetime]::TryParse($b, [ref]$db)
                    if ($okA -and $okB) { return ($da.ToUniversalTime() -eq $db.ToUniversalTime()) }
                    return $false
                  }

                  $live = Read-Settings $liveFile
                  $new  = Read-Settings $newFile
                  $allKeys = ($live.Keys + $new.Keys) | Sort-Object -Unique

                  $rows = foreach ($k in $allKeys) {
                    $inLive = $live.ContainsKey($k); $inNew = $new.ContainsKey($k)
                    if     (-not $inLive -and $inNew)            { [pscustomobject]@{ Action='ADDED';     Setting=$k; OldValue='';                       NewValue=(Truncate $new[$k] 45) } }
                    elseif ($inNew -and -not (ValuesEqual $live[$k] $new[$k])) { [pscustomobject]@{ Action='CHANGED';   Setting=$k; OldValue=(Truncate $live[$k] 45);  NewValue=(Truncate $new[$k] 45) } }
                    elseif ($inLive -and $inNew)                 { [pscustomobject]@{ Action='UNCHANGED'; Setting=$k; OldValue='';  NewValue='' } }
                    elseif ($inLive -and -not $inNew)            { [pscustomobject]@{ Action='LIVE-ONLY'; Setting=$k; OldValue='';  NewValue='' } }
                  }

                  Write-Host "Version To Be Deployed: ${{ parameters.artifactName }}"
                  Write-Host ""
                  Write-Host "App settings diff for ${{ env }}  (deploy MERGES: LIVE-ONLY rows are left untouched, nothing is deleted)"
                  Write-Host ""

                  function Show-Section($title, $items, $showValues) {
                    Write-Host ""
                    Write-Host "================ $title ================"
                    if (-not $items -or @($items).Count -eq 0) {
                      Write-Host "(none)"
                      return
                    }
                    if ($showValues) {
                      $items | Sort-Object Setting |
                        Format-Table -AutoSize Setting, @{Name='ExistingValue';Expression={$_.OldValue}}, @{Name='DesiredValue';Expression={$_.NewValue}} |
                        Out-String -Width 4096 | Write-Host
                    } else {
                      # values are secrets - show setting names only
                      $items | Sort-Object Setting |
                        Format-Table -AutoSize Setting |
                        Out-String -Width 4096 | Write-Host
                    }
                  }

                  if (-not $rows) { Write-Host "No settings found to compare." }
                  else {
                    Show-Section "CHANGED"   @($rows | Where-Object { $_.Action -eq 'CHANGED' })   $true
                    Show-Section "ADDED"     @($rows | Where-Object { $_.Action -eq 'ADDED' })     $true
                    Show-Section "LIVE-ONLY" @($rows | Where-Object { $_.Action -eq 'LIVE-ONLY' }) $false
                    Show-Section "UNCHANGED" @($rows | Where-Object { $_.Action -eq 'UNCHANGED' }) $false
                  }

                  $chg = @($rows | Where-Object { $_.Action -in 'ADDED','CHANGED' }).Count
                  $unc = @($rows | Where-Object { $_.Action -eq 'UNCHANGED' }).Count
                  $lo  = @($rows | Where-Object { $_.Action -eq 'LIVE-ONLY' }).Count
                  Write-Host ""
                  Write-Host "Summary: $chg to add/change, $unc unchanged, $lo live-only (untouched)."

        # ---- Single approval gate: review the diff above, then approve to deploy ----
        - job: diffApprovalJob
          dependsOn: prepJob
          displayName: Approve deploy after diff review (${{ env }})
          pool: server   # agentless - required for ManualValidation
          timeoutInMinutes: 60   # must be > the task timeout below (MS docs); task timeout is the real limit
          steps:
            - task: ManualValidation@0
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
            # re-establish version metadata + artifact in this job's workspace
            - ${{ if eq(parameters.deployCode, true) }}:
              - task: PowerShell@2
                displayName: Normalize version metadata
                inputs:
                  targetType: 'inline'
                  script: |
                    $versionParts = '${{ parameters.artifactName }}' -split '\-b'
                    $buildId = ($versionParts[-1] -replace '[^0-9]', '')
                    if (-not $buildId) { Write-Host "##vso[task.logissue type=error]Could not parse build id from artifact name '${{ parameters.artifactName }}'"; exit 1 }
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
                ${{ if eq(env, 'dev') }}:
                  serviceConnection: RxI-Dev2-Leap-05
                ${{ if eq(env, 'e2e-01') }}:
                  serviceConnection: RxI-NProd-05
                ${{ if eq(env, 'e2e-02') }}:
                  serviceConnection: RxI-E2E-02-Leap
                ${{ if eq(env, 'perf-01') }}:
                  serviceConnection: RxI-NProd-05
                ${{ if eq(env, 'prodfix-01') }}:
                  serviceConnection: RxI-PRODFIX-05
################################################ END NONPROD DEPLOYMENTS ################################################

  - stage: ProdDiff
    displayName: Prepare & Diff prod
    dependsOn: []   # runs immediately at trigger, same as nonprod envs
    variables:
      - name: prodFunctionAppName
        value: rxr-rxi-prod-01-cus-fa-purchaseordermanagement
      - name: prodAppSettingsFile
        value: '$(Build.SourcesDirectory)/temp/$(Build.BuildId)/prod-deploy/app-settings.json'
    jobs:
      # ---- Prepare & Diff prod (same pattern as the QE gate stage) ----
      - job: prepProdDiffJob
        displayName: Prepare & Diff prod
        pool: { name: $(poolName), demands: azureps }
        timeoutInMinutes: 60
        steps:
          - task: PowerShell@2
            name: buildAppSettings
            displayName: Build App Settings (prod)
            condition: eq(${{ parameters.deployAppSettings }}, true)
            inputs:
              targetType: 'inline'
              script: |
                New-Item -Path '$(prodAppSettingsFile)' -Value '' -Type File -Force
                & "$(Build.SourcesDirectory)/scripts/build-app-settings.ps1" -EnvName 'prod' -OutputPath '$(prodAppSettingsFile)'
              failOnStderr: true
              showWarnings: true
              pwsh: true
          - task: AzureCLI@2
            name: captureProdAppSettings
            displayName: "Capture existing app settings (prod)"
            condition: eq(${{ parameters.deployAppSettings }}, true)
            inputs:
              azureSubscription: Rxi-prod05-static-ui-RxR-SCM
              scriptType: pscore
              scriptLocation: inlineScript
              inlineScript: |
                $appName = "$(prodFunctionAppName)"
                $liveFile = "$(Build.SourcesDirectory)/temp/$(Build.BuildId)/prod-deploy/live-app-settings.json"
                New-Item -Path $liveFile -ItemType File -Force | Out-Null
                $rg = az functionapp list --query "[?name=='$appName'].resourceGroup | [0]" -o tsv
                if (-not $rg) {
                  Write-Host "##vso[task.logissue type=warning]Could not locate function app '$appName' in this subscription; skipping snapshot."
                  '[]' | Set-Content -Path $liveFile
                } else {
                  Write-Host "Resource group: $rg"
                  az functionapp config appsettings list --name $appName --resource-group $rg --output json | Set-Content -Path $liveFile
                }
          - task: PowerShell@2
            name: diffProdAppSettings
            displayName: "Diff app settings: live vs new (prod)"
            condition: eq(${{ parameters.deployAppSettings }}, true)
            inputs:
              targetType: 'inline'
              pwsh: true
              script: |
                $liveFile = "$(Build.SourcesDirectory)/temp/$(Build.BuildId)/prod-deploy/live-app-settings.json"
                $newFile  = "$(prodAppSettingsFile)"

                function Read-Settings($path) {
                  $map = @{}
                  if (-not (Test-Path $path)) { return $map }
                  $raw = Get-Content -Path $path -Raw
                  if ([string]::IsNullOrWhiteSpace($raw)) { return $map }
                  try { $json = $raw | ConvertFrom-Json } catch { return $map }
                  foreach ($item in @($json)) {
                    if ($null -eq $item) { continue }
                    if ($item.PSObject.Properties.Name -contains 'name') { $map[$item.name] = "$($item.value)" }
                    else { foreach ($prop in $item.PSObject.Properties) { $map[$prop.Name] = "$($prop.Value)" } }
                  }
                  return $map
                }

                function Truncate($s, $n) { if ($null -eq $s) { return '' }; if ($s.Length -le $n) { return $s }; return $s.Substring(0, $n-3) + '...' }

                function ValuesEqual($a, $b) {
                  if ($a -eq $b) { return $true }
                  $da = [datetime]::MinValue; $db = [datetime]::MinValue
                  $okA = [datetime]::TryParse($a, [ref]$da)
                  $okB = [datetime]::TryParse($b, [ref]$db)
                  if ($okA -and $okB) { return ($da.ToUniversalTime() -eq $db.ToUniversalTime()) }
                  return $false
                }

                $live = Read-Settings $liveFile
                $new  = Read-Settings $newFile
                $allKeys = ($live.Keys + $new.Keys) | Sort-Object -Unique

                $rows = foreach ($k in $allKeys) {
                  $inLive = $live.ContainsKey($k); $inNew = $new.ContainsKey($k)
                  if     (-not $inLive -and $inNew)            { [pscustomobject]@{ Action='ADDED';     Setting=$k; OldValue='';                       NewValue=(Truncate $new[$k] 45) } }
                  elseif ($inNew -and -not (ValuesEqual $live[$k] $new[$k])) { [pscustomobject]@{ Action='CHANGED';   Setting=$k; OldValue=(Truncate $live[$k] 45);  NewValue=(Truncate $new[$k] 45) } }
                  elseif ($inLive -and $inNew)                 { [pscustomobject]@{ Action='UNCHANGED'; Setting=$k; OldValue='';  NewValue='' } }
                  elseif ($inLive -and -not $inNew)            { [pscustomobject]@{ Action='LIVE-ONLY'; Setting=$k; OldValue='';  NewValue='' } }
                }

                Write-Host "Version To Be Deployed: ${{ parameters.artifactName }}"
                Write-Host ""
                Write-Host "App settings diff for prod  (deploy MERGES: LIVE-ONLY rows are left untouched, nothing is deleted)"
                Write-Host ""

                function Show-Section($title, $items, $showValues) {
                  Write-Host ""
                  Write-Host "================ $title ================"
                  if (-not $items -or @($items).Count -eq 0) { Write-Host "(none)"; return }
                  if ($showValues) {
                    $items | Sort-Object Setting |
                      Format-Table -AutoSize Setting, @{Name='ExistingValue';Expression={$_.OldValue}}, @{Name='DesiredValue';Expression={$_.NewValue}} |
                      Out-String -Width 4096 | Write-Host
                  } else {
                    $items | Sort-Object Setting | Format-Table -AutoSize Setting | Out-String -Width 4096 | Write-Host
                  }
                }

                if (-not $rows) { Write-Host "No settings found to compare." }
                else {
                  Show-Section "CHANGED"   @($rows | Where-Object { $_.Action -eq 'CHANGED' })   $true
                  Show-Section "ADDED"     @($rows | Where-Object { $_.Action -eq 'ADDED' })     $true
                  Show-Section "LIVE-ONLY" @($rows | Where-Object { $_.Action -eq 'LIVE-ONLY' }) $false
                  Show-Section "UNCHANGED" @($rows | Where-Object { $_.Action -eq 'UNCHANGED' }) $false
                }

                $chg = @($rows | Where-Object { $_.Action -in 'ADDED','CHANGED' }).Count
                $unc = @($rows | Where-Object { $_.Action -eq 'UNCHANGED' }).Count
                $lo  = @($rows | Where-Object { $_.Action -eq 'LIVE-ONLY' }).Count
                Write-Host ""
                Write-Host "Summary: $chg to add/change, $unc unchanged, $lo live-only (untouched)."

  - stage: ProdfixDiff
    displayName: Prepare & Diff prodfix-01
    dependsOn: []   # runs immediately at trigger, same as nonprod envs
    variables:
      - name: deployServiceConnection
        value: RxI-PRODFIX-05
      - name: functionAppName
        value: rxr-rxi-prodfix-01-cus-fa-purchaseordermanagement
      - name: appSettingsFile
        value: '$(Build.SourcesDirectory)/temp/$(Build.BuildId)/prodfix-01/app-settings.json'
    jobs:
      # ---- Prepare & Diff prodfix-01 (same pattern as prod) ----
      - job: prepProdfixDiffJob
        displayName: Prepare & Diff prodfix-01
        pool: { name: $(poolName), demands: azureps }
        timeoutInMinutes: 60
        steps:
          - task: PowerShell@2
            name: buildAppSettings
            displayName: Build App Settings (prodfix-01)
            condition: eq(${{ parameters.deployAppSettings }}, true)
            inputs:
              targetType: 'inline'
              script: |
                New-Item -Path '$(appSettingsFile)' -Value '' -Type File -Force
                & "$(Build.SourcesDirectory)/scripts/build-app-settings.ps1" -EnvName 'prodfix-01' -OutputPath '$(appSettingsFile)'
              failOnStderr: true
              showWarnings: true
              pwsh: true
          - task: AzureCLI@2
            name: captureProdfixAppSettings
            displayName: "Capture existing app settings (prodfix-01)"
            condition: eq(${{ parameters.deployAppSettings }}, true)
            inputs:
              azureSubscription: RxI-PRODFIX-05
              scriptType: pscore
              scriptLocation: inlineScript
              inlineScript: |
                $appName = "$(functionAppName)"
                $liveFile = "$(Build.SourcesDirectory)/temp/$(Build.BuildId)/prodfix-01/live-app-settings.json"
                New-Item -Path $liveFile -ItemType File -Force | Out-Null
                $rg = az functionapp list --query "[?name=='$appName'].resourceGroup | [0]" -o tsv
                if (-not $rg) {
                  Write-Host "##vso[task.logissue type=warning]Could not locate function app '$appName' in this subscription; skipping snapshot."
                  '[]' | Set-Content -Path $liveFile
                } else {
                  Write-Host "Resource group: $rg"
                  az functionapp config appsettings list --name $appName --resource-group $rg --output json | Set-Content -Path $liveFile
                }
          - task: PowerShell@2
            name: diffProdfixAppSettings
            displayName: "Diff app settings: live vs new (prodfix-01)"
            condition: eq(${{ parameters.deployAppSettings }}, true)
            inputs:
              targetType: 'inline'
              pwsh: true
              script: |
                $liveFile = "$(Build.SourcesDirectory)/temp/$(Build.BuildId)/prodfix-01/live-app-settings.json"
                $newFile  = "$(appSettingsFile)"

                function Read-Settings($path) {
                  $map = @{}
                  if (-not (Test-Path $path)) { return $map }
                  $raw = Get-Content -Path $path -Raw
                  if ([string]::IsNullOrWhiteSpace($raw)) { return $map }
                  try { $json = $raw | ConvertFrom-Json } catch { return $map }
                  foreach ($item in @($json)) {
                    if ($null -eq $item) { continue }
                    if ($item.PSObject.Properties.Name -contains 'name') { $map[$item.name] = "$($item.value)" }
                    else { foreach ($prop in $item.PSObject.Properties) { $map[$prop.Name] = "$($prop.Value)" } }
                  }
                  return $map
                }

                function Truncate($s, $n) { if ($null -eq $s) { return '' }; if ($s.Length -le $n) { return $s }; return $s.Substring(0, $n-3) + '...' }

                function ValuesEqual($a, $b) {
                  if ($a -eq $b) { return $true }
                  $da = [datetime]::MinValue; $db = [datetime]::MinValue
                  $okA = [datetime]::TryParse($a, [ref]$da)
                  $okB = [datetime]::TryParse($b, [ref]$db)
                  if ($okA -and $okB) { return ($da.ToUniversalTime() -eq $db.ToUniversalTime()) }
                  return $false
                }

                $live = Read-Settings $liveFile
                $new  = Read-Settings $newFile
                $allKeys = ($live.Keys + $new.Keys) | Sort-Object -Unique

                $rows = foreach ($k in $allKeys) {
                  $inLive = $live.ContainsKey($k); $inNew = $new.ContainsKey($k)
                  if     (-not $inLive -and $inNew)            { [pscustomobject]@{ Action='ADDED';     Setting=$k; OldValue='';                       NewValue=(Truncate $new[$k] 45) } }
                  elseif ($inNew -and -not (ValuesEqual $live[$k] $new[$k])) { [pscustomobject]@{ Action='CHANGED';   Setting=$k; OldValue=(Truncate $live[$k] 45);  NewValue=(Truncate $new[$k] 45) } }
                  elseif ($inLive -and $inNew)                 { [pscustomobject]@{ Action='UNCHANGED'; Setting=$k; OldValue='';  NewValue='' } }
                  elseif ($inLive -and -not $inNew)            { [pscustomobject]@{ Action='LIVE-ONLY'; Setting=$k; OldValue='';  NewValue='' } }
                }

                Write-Host "Version To Be Deployed: ${{ parameters.artifactName }}"
                Write-Host ""
                Write-Host "App settings diff for prodfix-01  (deploy MERGES: LIVE-ONLY rows are left untouched, nothing is deleted)"
                Write-Host ""

                function Show-Section($title, $items, $showValues) {
                  Write-Host ""
                  Write-Host "================ $title ================"
                  if (-not $items -or @($items).Count -eq 0) { Write-Host "(none)"; return }
                  if ($showValues) {
                    $items | Sort-Object Setting |
                      Format-Table -AutoSize Setting, @{Name='ExistingValue';Expression={$_.OldValue}}, @{Name='DesiredValue';Expression={$_.NewValue}} |
                      Out-String -Width 4096 | Write-Host
                  } else {
                    $items | Sort-Object Setting | Format-Table -AutoSize Setting | Out-String -Width 4096 | Write-Host
                  }
                }

                if (-not $rows) { Write-Host "No settings found to compare." }
                else {
                  Show-Section "CHANGED"   @($rows | Where-Object { $_.Action -eq 'CHANGED' })   $true
                  Show-Section "ADDED"     @($rows | Where-Object { $_.Action -eq 'ADDED' })     $true
                  Show-Section "LIVE-ONLY" @($rows | Where-Object { $_.Action -eq 'LIVE-ONLY' }) $false
                  Show-Section "UNCHANGED" @($rows | Where-Object { $_.Action -eq 'UNCHANGED' }) $false
                }

                $chg = @($rows | Where-Object { $_.Action -in 'ADDED','CHANGED' }).Count
                $unc = @($rows | Where-Object { $_.Action -eq 'UNCHANGED' }).Count
                $lo  = @($rows | Where-Object { $_.Action -eq 'LIVE-ONLY' }).Count
                Write-Host ""
                Write-Host "Summary: $chg to add/change, $unc unchanged, $lo live-only (untouched)."

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
      - name: prodFunctionAppName
        value: rxr-rxi-prod-01-cus-fa-purchaseordermanagement
      - name: prodAppSettingsFile
        value: '$(Build.SourcesDirectory)/temp/$(Build.BuildId)/prod/app-settings.json'
    jobs:
      # ---- Prepare & Diff for prod now runs INSIDE the QE gate stage ----
      - job: prepProdJob
        displayName: Prepare & Diff prod
        pool: { name: $(poolName), demands: azureps }
        timeoutInMinutes: 60
        steps:
          - task: PowerShell@2
            name: buildAppSettings
            displayName: Build App Settings (prod)
            condition: eq(${{ parameters.deployAppSettings }}, true)
            inputs:
              targetType: 'inline'
              script: |
                New-Item -Path '$(prodAppSettingsFile)' -Value '' -Type File -Force
                & "$(Build.SourcesDirectory)/scripts/build-app-settings.ps1" -EnvName 'prod' -OutputPath '$(prodAppSettingsFile)'
              failOnStderr: true
              showWarnings: true
              pwsh: true
          - task: AzureCLI@2
            name: captureProdAppSettings
            displayName: "Capture existing app settings (prod)"
            condition: eq(${{ parameters.deployAppSettings }}, true)
            inputs:
              azureSubscription: Rxi-prod05-static-ui-RxR-SCM
              scriptType: pscore
              scriptLocation: inlineScript
              inlineScript: |
                $appName = "$(prodFunctionAppName)"
                $liveFile = "$(Build.SourcesDirectory)/temp/$(Build.BuildId)/prod/live-app-settings.json"
                New-Item -Path $liveFile -ItemType File -Force | Out-Null
                $rg = az functionapp list --query "[?name=='$appName'].resourceGroup | [0]" -o tsv
                if (-not $rg) {
                  Write-Host "##vso[task.logissue type=warning]Could not locate function app '$appName' in this subscription; skipping snapshot."
                  '[]' | Set-Content -Path $liveFile
                } else {
                  Write-Host "Resource group: $rg"
                  az functionapp config appsettings list --name $appName --resource-group $rg --output json | Set-Content -Path $liveFile
                }
          - task: PowerShell@2
            name: diffProdAppSettings
            displayName: "Diff app settings: live vs new (prod)"
            condition: eq(${{ parameters.deployAppSettings }}, true)
            inputs:
              targetType: 'inline'
              pwsh: true
              script: |
                $liveFile = "$(Build.SourcesDirectory)/temp/$(Build.BuildId)/prod/live-app-settings.json"
                $newFile  = "$(prodAppSettingsFile)"

                function Read-Settings($path) {
                  $map = @{}
                  if (-not (Test-Path $path)) { return $map }
                  $raw = Get-Content -Path $path -Raw
                  if ([string]::IsNullOrWhiteSpace($raw)) { return $map }
                  try { $json = $raw | ConvertFrom-Json } catch { return $map }
                  foreach ($item in @($json)) {
                    if ($null -eq $item) { continue }
                    if ($item.PSObject.Properties.Name -contains 'name') { $map[$item.name] = "$($item.value)" }
                    else { foreach ($prop in $item.PSObject.Properties) { $map[$prop.Name] = "$($prop.Value)" } }
                  }
                  return $map
                }

                function Truncate($s, $n) { if ($null -eq $s) { return '' }; if ($s.Length -le $n) { return $s }; return $s.Substring(0, $n-3) + '...' }

                function ValuesEqual($a, $b) {
                  if ($a -eq $b) { return $true }
                  $da = [datetime]::MinValue; $db = [datetime]::MinValue
                  $okA = [datetime]::TryParse($a, [ref]$da)
                  $okB = [datetime]::TryParse($b, [ref]$db)
                  if ($okA -and $okB) { return ($da.ToUniversalTime() -eq $db.ToUniversalTime()) }
                  return $false
                }

                $live = Read-Settings $liveFile
                $new  = Read-Settings $newFile
                $allKeys = ($live.Keys + $new.Keys) | Sort-Object -Unique

                $rows = foreach ($k in $allKeys) {
                  $inLive = $live.ContainsKey($k); $inNew = $new.ContainsKey($k)
                  if     (-not $inLive -and $inNew)            { [pscustomobject]@{ Action='ADDED';     Setting=$k; OldValue='';                       NewValue=(Truncate $new[$k] 45) } }
                  elseif ($inNew -and -not (ValuesEqual $live[$k] $new[$k])) { [pscustomobject]@{ Action='CHANGED';   Setting=$k; OldValue=(Truncate $live[$k] 45);  NewValue=(Truncate $new[$k] 45) } }
                  elseif ($inLive -and $inNew)                 { [pscustomobject]@{ Action='UNCHANGED'; Setting=$k; OldValue='';  NewValue='' } }
                  elseif ($inLive -and -not $inNew)            { [pscustomobject]@{ Action='LIVE-ONLY'; Setting=$k; OldValue='';  NewValue='' } }
                }

                Write-Host "Version To Be Deployed: ${{ parameters.artifactName }}"
                Write-Host ""
                Write-Host "App settings diff for prod  (deploy MERGES: LIVE-ONLY rows are left untouched, nothing is deleted)"
                Write-Host ""

                function Show-Section($title, $items, $showValues) {
                  Write-Host ""
                  Write-Host "================ $title ================"
                  if (-not $items -or @($items).Count -eq 0) { Write-Host "(none)"; return }
                  if ($showValues) {
                    $items | Sort-Object Setting |
                      Format-Table -AutoSize Setting, @{Name='ExistingValue';Expression={$_.OldValue}}, @{Name='DesiredValue';Expression={$_.NewValue}} |
                      Out-String -Width 4096 | Write-Host
                  } else {
                    $items | Sort-Object Setting | Format-Table -AutoSize Setting | Out-String -Width 4096 | Write-Host
                  }
                }

                if (-not $rows) { Write-Host "No settings found to compare." }
                else {
                  Show-Section "CHANGED"   @($rows | Where-Object { $_.Action -eq 'CHANGED' })   $true
                  Show-Section "ADDED"     @($rows | Where-Object { $_.Action -eq 'ADDED' })     $true
                  Show-Section "LIVE-ONLY" @($rows | Where-Object { $_.Action -eq 'LIVE-ONLY' }) $false
                  Show-Section "UNCHANGED" @($rows | Where-Object { $_.Action -eq 'UNCHANGED' }) $false
                }

                $chg = @($rows | Where-Object { $_.Action -in 'ADDED','CHANGED' }).Count
                $unc = @($rows | Where-Object { $_.Action -eq 'UNCHANGED' }).Count
                $lo  = @($rows | Where-Object { $_.Action -eq 'LIVE-ONLY' }).Count
                Write-Host ""
                Write-Host "Summary: $chg to add/change, $unc unchanged, $lo live-only (untouched)."

      # ---- QE approval now gated on the prod diff above, within the same stage ----
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
        value: "Rxi-prod05-static-ui-RxR-SCM"
      - name: functionAppName
        value: "rxr-rxi-prod-cus-fa-purchaseordermanagement"
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
          #     serviceConnection: Rxi-prod05-static-ui-RxR-SCM
          #     healthcheckEnabled: false
          #     functionAppDeployEnabled: ${{ parameters.deployCode }}
          #     functionAppSettingsPublishEnabled: ${{ parameters.deployAppSettings }}
################################################ END PROD DEPLOYMENT STAGE ################################################
################################################ START PRODFIX DEPLOYMENT STAGES ################################################
  - stage: post_deploy_stage_prodfix_01
    displayName: "Deploy to prodfix-01"
    dependsOn: DeployToProd
    condition: succeeded('DeployToProd')
    variables:
      - name: deployServiceConnection
        value: RxI-PRODFIX-05
      - name: functionAppName
        value: rxr-rxi-prodfix-01-cus-fa-purchaseordermanagement
      - name: appSettingsFile
        value: '$(Build.SourcesDirectory)/temp/$(Build.BuildId)/prodfix-01/app-settings.json'
    jobs:
      - template: jobs/job-manual-approval.yml@rxiPipelineTemplate
        parameters:
          jobName: prodfix_approval
          timeoutInMinutes: 10
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
                pipelineId: '$(pipelineBuildId)'
                artifactName: '$(artifactName)'
                itemPattern: '**'
                targetPath: '$(downloadedArtifacts)/prodfix-01'
          - pwsh: |
              Write-Host "Artifact download complete for prodfix-01 (deployment steps are commented out)"
            displayName: "Confirm artifact download"
          # - task: PowerShell@2
          #   name: buildAppSettings
          #   displayName: Build App Settings
          #   condition: eq(${{ parameters.deployAppSettings }}, true)
          #   inputs:
          #     targetType: 'inline'
          #     script: |
          #       New-Item -Path '$(appSettingsFile)' -Value '' -Type File -Force
          #       & "$(Build.SourcesDirectory)/scripts/build-app-settings.ps1" -EnvName 'prodfix-01' -OutputPath '$(appSettingsFile)'
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
          #     package: '$(downloadedArtifacts)/prodfix-01/**/*${{ parameters.artifactName }}*.zip'
          #     serviceConnection: RxI-PRODFIX-05
          #     healthcheckEnabled: false
          #     functionAppDeployEnabled: ${{ parameters.deployCode }}
          #     functionAppSettingsPublishEnabled: ${{ parameters.deployAppSettings }}
################################################ END PRODFIX DEPLOYMENT STAGES ################################################

```
{% endraw %}
