{% raw %}
```yaml


steps:
  - task: JFrogMaven@1
    displayName: 'Build Maven and Deploy to JFrog'
    inputs:
      mavenPomFile: 'pom.xml'
      goals: 'clean deploy'

      artifactoryResolverService: 'jfrog-artifactory-v2'
      targetResolveReleaseRepo: 'libs-release'
      targetResolveSnapshotRepo: 'libs-snapshot'

      artifactoryDeployService: 'jfrog-artifactory-v2'
      targetDeployReleaseRepo: 'libs-release-local'
      targetDeploySnapshotRepo: 'libs-snapshot-local'

      collectBuildInfo: true
      buildName: '$(Build.DefinitionName)'
      buildNumber: '$(Build.BuildNumber)'





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
- It runs a Maven build and connects that build to JFrog Artifactory


### Replacement 
- ArtifactoryMaven@1 has an official replacement in Azure DevOps: JFrogMaven@1.
  https://marketplace.visualstudio.com/items?itemName=JFrog.jfrog-artifactory-vsts-extension&ssr=false
