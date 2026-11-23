{% raw %}
```yaml

steps:
- task: NodeTool@0
  displayName: Use Node 24
  inputs:
    versionSpec: '24.x'

- script: |
    corepack enable
    yarn --version
  displayName: Enable Corepack (Yarn from packageManager)

- script: yarn install --immutable
  displayName: yarn install

- script: yarn build
  displayName: yarn build


```
{% endraw %}

```bash


```

{% raw %}
```yaml


```
{% endraw %}


- Yarn@3 task reference

What the task does

It's a thin wrapper around the yarn CLI. It runs a Yarn command, such as install or run build, inside your repo. It was modeled on the built-in npm task, so it usually offers these conveniences:

- Command and arguments: you pass the command to run and its flags.
- Working directory: it runs in a chosen project folder, which matters in monorepos.
- Registry and auth: it can point at an Azure Artifacts feed or an external registry through a service connection. It handles the .npmrc credentials so you don't have to.
- Log and error handling: it fails the step on a non-zero exit code and writes its output to the pipeline log.
