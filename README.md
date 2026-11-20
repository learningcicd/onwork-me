{% raw %}
```yaml
- bash: |
    echo "##vso[build.updatebuildnumber]$(Build.SourceBranchName)-$(Build.BuildId)"
  displayName: Update build number

- powershell: |
    Write-Host "##vso[build.updatebuildnumber]1.0.$(Build.BuildId)"
  displayName: Update build number




```
{% endraw %}

```bash
https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/publish-pipeline-artifact-v1?view=azure-pipelines

```

{% raw %}
```yaml
trigger: none

variables:
  major: 1
  minor: 0

stages:
  - stage: NodeWarningCheck
    variables:
      AGENT_RESTRICT_EOL_NODE_VERSIONS: 'true'
      AGENT_DISABLE_NODE6_TASKS: 'true'
    jobs:
      - job: UpdateBuildNumberTest
        pool:
          name: 'agent-for-node-runner-test'
        steps:
          - bash: echo "Before update -> $(Build.BuildNumber)"
            displayName: 1. Show original build number

          - bash: |
              NEW="$(major).$(minor).$(Build.BuildId)-$(Build.SourceBranchName)"
              echo "Setting build number to: $NEW"
              echo "##vso[build.updatebuildnumber]$NEW"
            displayName: 2. Update build number (replaces UpdateBuildNumber@0)

          - bash: |
              echo "After update  -> $(Build.BuildNumber)"
              EXPECTED="$(major).$(minor).$(Build.BuildId)-$(Build.SourceBranchName)"
              if [ "$(Build.BuildNumber)" != "$EXPECTED" ]; then
                echo "##vso[task.logissue type=error]Build number mismatch: got '$(Build.BuildNumber)', expected '$EXPECTED'"
                exit 1
              fi
              echo "PASS: build number updated correctly"
            displayName: 3. Verify new build number

```
{% endraw %}

- https://learn.microsoft.com/en-us/azure/devops/pipelines/scripts/logging-commands?view=azure-devops
- 
