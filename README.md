{% raw %}
```yaml
trigger: none
#trigger Monitoring RU
#trigger montly on Monday at italian time: 3:00.
schedules:
  - cron: "0 1 * * 1"
    always: true
    displayName: Monitoring RU
    branches:
      include:
        - master

parameters:
  - name: publishReports
    displayName: "Publish Report"
    type: boolean
    default: true
  - name: isSendMail
    displayName: "Send Alert Email"
    type: boolean
    default: true

pool:
  name: "<NONPROD_AGENT_POOL_NAME>"          # e.g. rxi-be17-agent-pool-nprod

resources:
  repositories:
    - repository: RepoWikiMonitoring
      type: git
      name: "RxR-Architecture/rxi-monitoring-wiki"
      ref: refs/heads/master

variables:
  - group: RxI-Monitoring-Naples
  - group: CIsharedvariables
  - name: envName
    value: "nonprod"
  - name: StorageAccountName
    value: 'rxrrxiprf01cusst'
  - name: serviceConnectionNameStorage
    value: "RxI-PERF-05-BLOB"
  - name: ContainerName
    value: 'rxi-monitoring-reports'

stages:
  - stage: StageMonitoringPhysicalPartitionOverSize
    displayName: "Monitoring RU"
    variables:
      - name: repositoryMonitoringPath
        value: "$(System.DefaultWorkingDirectory)/rxi-utils/cosmos"
      - name: repositoryMonitoringUtilsPath
        value: "$(System.DefaultWorkingDirectory)/rxi-utils/utils"
      - name: reportsFolder
        value: "$(System.DefaultWorkingDirectory)/cosmos/reports"
      - name: wikiBuildFolder
        value: "$(Build.SourcesDirectory)/cosmos/reports/wiki"
      - name: wikiADOFolder
        value: "$(System.DefaultWorkingDirectory)/rxi-monitoring-wiki/lower-env/CosmosDB"
      - name: accountName
        value: "<NONPROD_COSMOS_ACCOUNT_NAME>"          # e.g. rxr-rxi-perf-cus-cosmos-01
      - name: resourceGroup
        value: "<NONPROD_RESOURCE_GROUP_NAME>"          # e.g. rxr-rxi-perf-cus-rg
      - name: subscriptionName
        value: "<NONPROD_SUBSCRIPTION_NAME>"            # e.g. rpu-nprod-rxrenewal-01
      - name: subscriptionId
        value: "<NONPROD_SUBSCRIPTION_ID>"
      - name: subscriptionCosmosId
        value: "<NONPROD_SUBSCRIPTION_COSMOS_ID>"
      - name: nameBlobReport
        value: "report-$(accountName)-RU.csv"
      - name: FullBlobReport
        value: "cosmos/$(nameBlobReport)"
    jobs:
      - job: RunningMonitoring
        timeoutInMinutes: 480
        steps:
          - checkout: self
            fetchDepth: 0
          - checkout: RepoWikiMonitoring
            clean: true
            persistCredentials: true
          - task: PowerShell@2
            displayName: "Cleaning Artifacts Folder"
            inputs:
              targetType: 'inline'
              script: |
                $artifactFile = "${{ variables.reportsFolder }}"
                if (Test-Path $artifactFile) { Remove-Item -Recurse -Force $artifactFile }
          - task: AzureCLI@2
            displayName: 'Download From Blob Storage'
            condition: eq(${{ parameters['publishReports'] }}, True)
            continueOnError: true
            inputs:
              azureSubscription: '$(serviceConnectionNameStorage)'
              scriptType: 'pscore'
              scriptLocation: 'scriptPath'
              scriptPath: '${{ variables.repositoryMonitoringUtilsPath }}/download-blobs.ps1'
              arguments: "-AccountName $(StorageAccountName) -Container $(ContainerName) -ResourceGroup '$(storageAccountResourceGroup)'
              -BlobName '$(FullBlobReport)'
              -DestinationPath '$(repositoryMonitoringPath)'"
          - task: AzureCLI@2
            displayName: "Generate Reports"
            inputs:
              azureSubscription: "<NONPROD_SERVICE_CONNECTION_NAME>"          # e.g. RxI-E2E-Leap-05
              scriptType: "pscore"
              scriptLocation: "inlineScript"
              inlineScript: |
                cd '${{ variables.repositoryMonitoringPath }}'
                $password = ConvertTo-SecureString "$(principalKEY)" -AsPlainText -Force
                $SecretCodeFuncMailSenderEncoded = ConvertTo-SecureString "$(SecretCodeFuncMailSender)" -AsPlainText -Force
                $Credential = New-Object System.Management.Automation.PsCredential("$(principalID)", $password)
                $tenantId = "$(tenantId)"
                Connect-AzAccount -ServicePrincipal -TenantId $tenantId -Credential $Credential -SubscriptionId ${{ variables.subscriptionName }}
                $Params = @{
                    "RunnedBy"                 = "$(Build.QueuedBy)"
                    "NamePipeline"              = "$(Build.DefinitionName)"
                    "IdPipeline"                = "$(Build.BuildId)"
                    "envName"                   = "${{ variables.envName }}"
                    "accountName"               = "${{ variables.accountName }}"
                    "resourceGroup"             = "${{ variables.resourceGroup }}"
                    "subscriptionName"          = "${{ variables.subscriptionName }}"
                    "subscriptionCosmosId"      = "${{ variables.subscriptionCosmosId }}"
                    "subscriptionId"            = "${{ variables.subscriptionId }}"
                    "TargetFolder"              = "${{ variables.reportsFolder }}"
                    "publishReports"            = "${{ parameters.publishReports }}"
                    "isSendMail"                = "${{ parameters.isSendMail }}"
                    "SecretCodeFuncMailSender"  = $SecretCodeFuncMailSenderEncoded
                    "ToMail"                    = "DEFAULT"
                    "CcMail"                    = "DEFAULT"
                }
                & "./monitor-ru.ps1" @Params
          - task: PublishBuildArtifacts@1
            displayName: "Publish Artifact"
            inputs:
              PathtoPublish: "$(reportsFolder)"
              ArtifactName: drop
              publishLocation: "Container"
          - task: AzureCLI@2
            displayName: 'Publish on Blob Storage'
            condition: eq(${{ parameters['publishReports'] }}, True)
            inputs:
              azureSubscription: '$(serviceConnectionNameStorage)'
              scriptType: 'pscore'
              scriptLocation: 'scriptPath'
              scriptPath: '${{ variables.repositoryMonitoringUtilsPath }}/publish-blobs.ps1'
              arguments: "-AccountName $(StorageAccountName) -Container $(ContainerName) -ResourceGroup '$(storageAccountResourceGroup)' -Source '$(reportsFolder)/$(FullBlobReport)' -RelativeBasePath '$(reportsFolder)' -Target ''"
          - bash: |
              cd ${{ variables.wikiADOFolder }}
              git config --global user.name "AzureDevOpsUser"
              git config --global user.email azureDevOpsUser@walgreens.com
              git fetch origin master
              git checkout -B master origin/master
              mv "${{ variables.wikiBuildFolder }}"/*.MD "${{ variables.wikiADOFolder }}"
              git add --all
              git commit -m "Update ADO"
              git push origin master
            displayName: "Update Wiki ADO"
            condition: eq(${{ parameters['publishReports'] }}, True)
            retryCountOnTaskFailure: 3
            continueOnError: true

```
{% endraw %}



