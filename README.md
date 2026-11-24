{% raw %}
```yaml

steps:
  - task: JFrogPublishBuildInfo@1
    displayName: 'Publish JFrog Build Info'
    inputs:
      artifactoryConnection: 'jfrog-artifactory-v2'
      buildName: '$(Build.DefinitionName)'
      buildNumber: '$(Build.BuildNumber)'
      excludeEnvVars: '*password*;*psw*;*secret*;*key*;*token*;*auth*;'





```
{% endraw %}

```bash


```

{% raw %}
```yaml


```
{% endraw %}


- ArtifactoryPublishBuildInfo@1

What the task does
- It sends a record of your build to JFrog Artifactory. It collects data like were uploaded, which dependencies were resolved, and details about the environment and posts it to Artifactory as a single "build-info" entry.

### Replacement 
- https://docs.jfrog.com/integrations/docs/jfrog-azure-devops-extension
- https://marketplace.visualstudio.com/items?itemName=JFrog.jfrog-azure-devops-extension
