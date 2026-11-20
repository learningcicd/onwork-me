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

pool:
  vmImage: ubuntu-latest   # use your real pool/agent to confirm the node24 warning is gone

parameters:
- name: failBuild
  type: boolean
  default: false

steps:
- bash: |
    mkdir -p "$(Build.SourcesDirectory)/reports"
    echo '<html><body><h1>Test report</h1><p>$(Build.BuildNumber)</p></body></html>' > "$(Build.SourcesDirectory)/reports/report.html"
  displayName: Generate dummy report

- bash: |
    if [ "${{ parameters.failBuild }}" = "True" ]; then exit 1; fi
  displayName: Optional forced failure

# --- step under test ---
- task: PublishPipelineArtifact@1
  displayName: Publish HTML report
  condition: succeededOrFailed()
  inputs:
    targetPath: $(Build.SourcesDirectory)/reports/report.html
    artifact: html-report-$(System.JobAttempt)

```
{% endraw %}
