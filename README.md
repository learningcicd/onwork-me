{% raw %}
```yaml
- task: HelmInstaller@1        # optional, pins the Helm version
  inputs:
    helmVersionToInstall: 'latest'

- task: HelmDeploy@1
  inputs:
    connectionType: 'Azure Resource Manager'
    azureSubscription: '<service connection>'
    azureResourceGroup: '<rg>'
    kubernetesCluster: '<aks>'
    namespace: '<ns>'
    command: 'upgrade'
    chartType: 'FilePath'
    chartPath: './charts/myapp'
    releaseName: 'myapp'
    install: true

```
{% endraw %}

```bash

```
