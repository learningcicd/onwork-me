{% raw %}
```yaml

steps:
  - task: JfrogCliV2@1
    displayName: 'Verify JFrog Artifactory'
    inputs:
      jfrogPlatformConnection: 'jfrog-platform-prod'
      command: 'jf rt ping'


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
https://marketplace.visualstudio.com/items?itemName=JFrog.jfrog-azure-devops-extension&utm_source=chatgpt.com
