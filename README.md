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
