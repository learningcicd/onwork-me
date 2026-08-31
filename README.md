{% raw %}
```yaml
trigger: none
#trigger daily cosmos report generation
#trigger every Monday at italian time: 11:00
schedules:
  - cron: "0 9 * * 1" #utc time
    always: true
    displayName: Daily cosmos report generation for PowerBI
    branches:
      include:
      - master

pool:
  name: rxi-agent-pool

parameters:

  - name: containerFilter
    displayName: "Container Filter (Glob pattern)"
    type: string
    default: '*'

  - name: publishReports
    displayName: "Publish Reports"
    type: boolean
    default: true

variables:
- group: CIsharedvariables
- name: account-name
  value: "<NONPROD_COSMOS_ACCOUNT_NAME>"
- name: resource-group
  value: "<NONPROD_RESOURCE_GROUP>"
- name: serviceConnectionNameCosmos
  value: "<NONPROD_SERVICE_CONNECTION_COSMOS>"
- name: subscriptionName
  value: "<NONPROD_SUBSCRIPTION_NAME>"
- name: StorageAccountName
  value: 'rxrrxiprf01cusst'
- name: ContainerName
  value: 'rxi-monitoring-cosmos'
- name: publishReports
  value: ${{ parameters.publishReports }}
- name: containerFilter
  value: '${{ parameters.containerFilter }}'
- name: reportsFolder
  value: '$(System.DefaultWorkingDirectory)/monitoring/cosmos/generate-cosmos-reports/reports'
- name: serviceConnectionNameStorage
  value: "RxI-PERF-05-BLOB"
- name: storageAccountResourceGroup
  value: "rxr-rxi-perf-01-cus-rg"

stages:

- stage: Generate_Report
  jobs:

  - job: GenerateReport
    timeoutInMinutes: 480
    steps:
    - checkout: self
      fetchDepth: 0
    - task: AzureCLI@2
      displayName: 'Generate Reports'
      inputs:
        azureSubscription: '$(serviceConnectionNameCosmos)'
        scriptType: 'pscore'
        scriptLocation: 'scriptPath'
        scriptPath: '$(System.DefaultWorkingDirectory)/monitoring/cosmos/generate-cosmos-reports/runner.ps1'
        arguments: "-AccountName $(account-name) -ResourceGroup $(resource-group) -Subscription $(subscriptionName)
        -TargetFolder '$(reportsFolder)' -ContainerFilter '$(containerFilter)'"

    - task: PublishBuildArtifacts@1
      displayName: 'Publish Artifact'
      inputs:
        PathtoPublish: '$(reportsFolder)'
        ArtifactName: drop
        publishLocation: 'Container'

    - task: AzureCLI@2
      displayName: 'Publish on Blob Storage'
      condition: eq(variables.publishReports, 'true')
      inputs:
        azureSubscription: '$(serviceConnectionNameStorage)'
        scriptType: 'pscore'
        scriptLocation: 'scriptPath'
        scriptPath: '$(System.DefaultWorkingDirectory)/monitoring/utils/publish-blobs.ps1'
        arguments: "-AccountName $(StorageAccountName) -Container $(ContainerName)
        -ResourceGroup '$(storageAccountResourceGroup)' -Source '$(reportsFolder)/*' -RelativeBasePath '$(reportsFolder)'
        -Target ''"

```
{% endraw %}


