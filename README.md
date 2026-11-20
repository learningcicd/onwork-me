{% raw %}
```yaml
- task: PublishPipelineArtifact@1
  displayName: Publish HTML report
  condition: succeededOrFailed()
  inputs:
    targetPath: $(Build.SourcesDirectory)/reports/html   # your report folder
    artifact: html-report-$(System.JobAttempt)

- bash: |
    cat > "$(Agent.TempDirectory)/html-report.md" <<'EOF'
    ## HTML report
    Open the **Artifacts** list on this run, download `html-report-$(System.JobAttempt)`, and open `index.html`.
    EOF
    echo "##vso[task.uploadsummary]$(Agent.TempDirectory)/html-report.md"
  displayName: Add report summary
  condition: succeededOrFailed()

```
{% endraw %}

```bash
https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/publish-pipeline-artifact-v1?view=azure-pipelines

```
