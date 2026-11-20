{% raw %}
```yaml
- task: PublishPipelineArtifact@1
  displayName: Publish HTML report
  condition: succeededOrFailed()
  inputs:
    targetPath: $(Build.SourcesDirectory)/reports/report.html   # your htmlFilePath value
    artifact: html-report-$(System.JobAttempt)

```
{% endraw %}

```bash
https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/publish-pipeline-artifact-v1?view=azure-pipelines

```

{% raw %}
```yaml
trigger: none

stages:
- stage: NodeWarningCheck
  variables:
    AGENT_RESTRICT_EOL_NODE_VERSIONS: 'true'
    AGENT_DISABLE_NODE6_TASKS: 'true'
  jobs:
  - job: PublishHtmlTest
    pool:
      name: 'agent-for-node-runner-test'
    steps:
    - bash: |
        mkdir -p "$(Build.SourcesDirectory)/reports"
        echo '<html><body><h1>Test report</h1><p>$(Build.BuildNumber)</p></body></html>' > "$(Build.SourcesDirectory)/reports/report.html"
      displayName: Generate dummy report

    - task: PublishPipelineArtifact@1
      displayName: Publish HTML report
      condition: succeededOrFailed()
      inputs:
        targetPath: $(Build.SourcesDirectory)/reports/report.html
        artifact: html-report-$(System.JobAttempt)

```
{% endraw %}
