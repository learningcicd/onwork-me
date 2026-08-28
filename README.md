{% raw %}
```yaml
trigger: none
#trigger Physical Partition Over Size
#trigger at italian times: 5:00 on every Monday
schedules:
  - cron: "0 3 * * 1"
    always: true
    displayName: Physical Partition Over Size
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
  #name: rxi-be17-agent-pool-prod
  name: "rxi-be17-agent-pool"

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
  - name: sizeGB
    value: "40"
  - name: StorageAccountName
    value: 'rxrrxiprf01cusst'
  - name: serviceConnectionNameStorage
    value: "RxI-PERF-05-BLOB"
  - name: storageAccountResourceGroup
    value: "rxr-rxi-perf-01-cus-rg"
  - name: ContainerName
    value : 'rxi-monitoring-reports'

stages:
  - stage: StageMonitoringPhysicalPartitionOverSize
    displayName: "Monitoring Physical Partition Over Size"
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
        value: "rxi-perf-nprod-cus"
      - name: nameBlobReport
        value: "cosmos/report-$(accountName)-Partition-Over-($(sizeGB))GB.csv"
      - name: resourceGroup
        value: "rxr-rxi-perf-01-cus-rg"
      - name: subscriptionName
        value: "rpu-nprod-rxrenewal-05"
      - name: subscriptionId
        value: "117cc72c-bb3d-4b86-beba-be946bf2f5d8"
      - name: subscriptionCosmosId
        value: "d6e2435c-9500-4114-8724-fb0cad5d50b7"
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
            displayName: "Generate Reports"
            inputs:
              azureSubscription: "RxI-NProd-05"
              scriptType: "pscore"
              scriptLocation: "inlineScript"
              inlineScript: |
                cd '${{ variables.repositoryMonitoringPath }}'
                $username = "$(bamboocheckoutuser)"
                $passwordConfluence = ConvertTo-SecureString "$(bamboocheckoutpass)" -AsPlainText -Force
                $credentialConfluence = new-object -typename System.Management.Automation.PSCredential -argumentlist $username, $passwordConfluence
                $password = ConvertTo-SecureString "$(principalKEY)" -AsPlainText -Force
                $SecretCodeFuncMailSenderEncoded = ConvertTo-SecureString "$(SecretCodeFuncMailSender)" -AsPlainText -Force
                $Credential = New-Object System.Management.Automation.PsCredential("$(principalID)",$password)
                $tenantId="$(tenantId)"
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
                    "Size"                      = "${{ variables.sizeGB }}"
                    "SeverityLevel"             = "2"
                    "TargetFolder"              = "${{ variables.reportsFolder }}"
                    "publishReports"            = "${{ parameters.publishReports }}"
                    "isSendMail"                = "${{ parameters.isSendMail }}"
                    "SecretCodeFuncMailSender"  = $SecretCodeFuncMailSenderEncoded
                    "ToMail"                    = "DEFAULT"
                    "CcMail"                    = "DEFAULT"
                    "ConfluenceSpace"           = "~fzivient"
                    "ParentPageId"              = "238467051"
                    "Credential"                = $credentialConfluence
                }
                & "./physical-partition-over-size.ps1" @Params
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
              arguments: "-AccountName $(StorageAccountName) -Container $(ContainerName) -ResourceGroup '$(storageAccountResourceGroup)' -Source '$(reportsFolder)/$(nameBlobReport)' -RelativeBasePath '$(reportsFolder)' -Target ''"
          - bash: |
              cd ${{ variables.wikiADOFolder }}
              git config --global user.name "AzureDevOpsUser"
              git config --global user.email azureDevOpsUser@walgreens.com
              git switch master
              git pull origin master --rebase
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


