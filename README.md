{% raw %}
```yaml


steps:
  - bash: |
      mkdir -p "$(Build.ArtifactStagingDirectory)"
      echo "JFrog Upload Test" > "$(Build.ArtifactStagingDirectory)/test.txt"
    displayName: 'Create Test Artifact'

  - task: JFrogGenericArtifacts@1
    displayName: 'Upload Artifact to JFrog'
    inputs:
      command: 'Upload'
      connection: 'jfrog-artifactory-v2'
      specSource: 'taskConfiguration'
      fileSpec: |
        {
          "files": [
            {
              "pattern": "$(Build.ArtifactStagingDirectory)/test.txt",
              "target": "generic-local/ado-test/$(Build.BuildId)/"
            }
          ]
        }
      collectBuildInfo: true
      buildName: '$(Build.DefinitionName)'
      buildNumber: '$(Build.BuildNumber)'
      failNoOp: true



```
{% endraw %}

```bash


```

{% raw %}
```yaml


```
{% endraw %}


- ArtifactoryGenericUpload@2

What the task does

It uploads files from the build agent to a repository in JFrog Artifactory, using the JFrog CLI under the hood

## Unmiantained
https://marketplace.visualstudio.com/items?itemName=JFrog.jfrog-artifactory-vsts-extension

### Replacement 
- https://marketplace.visualstudio.com/items?itemName=JFrog.jfrog-azure-devops-extension&utm_source=chatgpt.com
- https://docs.jfrog.com/integrations/docs/jfrog-azure-devops-extension
