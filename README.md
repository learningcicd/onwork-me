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

# Initial run name (replaced by the step below)
name: initial-$(Date:yyyyMMdd)$(Rev:.r)

pool:
  vmImage: ubuntu-latest   # or your self-hosted pool

variables:
  major: 1
  minor: 0

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

  # Optional: Windows agent check
  - powershell: |
      Write-Host "##vso[build.updatebuildnumber]$(major).$(minor).$(Build.BuildId)-ps"
    displayName: 4. (Optional) PowerShell variant
    condition: eq(variables['Agent.OS'], 'Windows_NT')

```
{% endraw %}

- https://learn.microsoft.com/en-us/azure/devops/pipelines/scripts/logging-commands?view=azure-devops
- 
