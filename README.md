{% raw %}
```yaml
# Run each task once on a Node 24 agent and check for the Node.js EOL warning:
#   "Task 'X' version N (X@N) is dependent on a Node version (NN) that is end-of-life"
# Every step has continueOnError: true - the warning is emitted when the agent
# picks the Node handler, before the task runs, so task failures don't matter.
# Azure tasks point at a resource group/app that does NOT exist, so nothing changes.

trigger: none
pr: none

pool:
  name: 'Node24-SelfHosted'          # <-- your Node 24 agent pool

variables:
  system.debug: true                 # agent logs which node binary it used

steps:
- checkout: self
  displayName: 'Checkout@1'

- bash: |
    ls -1 "$(Agent.HomeDirectory)/externals" | grep -i '^node'
    "$(Agent.HomeDirectory)/externals/node24/bin/node" --version
  displayName: 'Verify Node 24 on agent'

- task: AzureAppServiceSettings@1
  continueOnError: true
  inputs:
    azureSubscription: 'my-arm-service-connection'
    appName: 'app-node24check-does-not-exist'
    resourceGroupName: 'rg-node24check-does-not-exist'
    appSettings: '[{"name":"NODE24_CHECK","value":"1","slotSetting":false}]'

- task: AzureCLI@2
  continueOnError: true
  inputs:
    azureSubscription: 'my-arm-service-connection'
    scriptType: bash
    scriptLocation: inlineScript
    inlineScript: az version

- task: AzureFunctionApp@2
  continueOnError: true
  inputs:
    connectedServiceNameARM: 'my-arm-service-connection'
    appType: functionAppLinux
    appName: 'app-node24check-does-not-exist'
    package: '$(System.DefaultWorkingDirectory)/**/*.zip'

- task: AzurePowerShell@5
  continueOnError: true
  inputs:
    azureSubscription: 'my-arm-service-connection'
    azurePowerShellVersion: LatestVersion
    ScriptType: InlineScript
    Inline: 'Get-AzContext'
    pwsh: true

- task: Bash@3
  continueOnError: true
  inputs:
    targetType: inline
    script: echo "Bash@3 ran"

- task: CmdLine@2
  continueOnError: true
  inputs:
    script: echo "CmdLine@2 ran"

- task: CopyFiles@2
  continueOnError: true
  inputs:
    SourceFolder: '$(System.DefaultWorkingDirectory)'
    Contents: '*.yml'
    TargetFolder: '$(Build.ArtifactStagingDirectory)'

- task: Docker@2
  continueOnError: true
  inputs:
    command: build
    repository: node24check/probe
    Dockerfile: '**/Dockerfile'
    tags: '$(Build.BuildId)'

- task: DotNetCoreCLI@2
  continueOnError: true
  inputs:
    command: custom
    custom: '--info'

- task: PublishBuildArtifacts@1
  continueOnError: true
  inputs:
    PathtoPublish: '$(Build.ArtifactStagingDirectory)'
    ArtifactName: node24probe

- task: DownloadPipelineArtifact@2
  continueOnError: true
  inputs:
    buildType: current
    artifactName: node24probe
    targetPath: '$(Pipeline.Workspace)/node24probe'

- task: HelmDeploy@0
  continueOnError: true
  inputs:
    connectionType: None
    command: ls

- task: MicrosoftSecurityDevOps@1
  continueOnError: true
  inputs:
    tools: templateanalyzer

- task: PowerShell@2
  continueOnError: true
  inputs:
    targetType: inline
    script: '$PSVersionTable.PSVersion'
    pwsh: true

- task: PythonScript@0
  continueOnError: true
  inputs:
    scriptSource: inline
    script: print("PythonScript@0 ran")

- task: UseDotNet@2
  continueOnError: true
  inputs:
    packageType: sdk
    version: 8.x

- task: Kubernetes@1
  continueOnError: true
  inputs:
    connectionType: None
    command: version
    arguments: '--client'

# Print one line per task: EOL warning (and Node version) or CLEAN.
- bash: |
    curl -s -H "Authorization: Bearer $SYSTEM_ACCESSTOKEN" \
      "$(System.CollectionUri)$(System.TeamProject)/_apis/build/builds/$(Build.BuildId)/timeline?api-version=7.1" \
      -o timeline.json
    python3 - <<'EOF'
    import json, re
    recs = [r for r in json.load(open("timeline.json"))["records"]
            if r["type"] == "Task" and r.get("task")]
    print("%-40s %s" % ("TASK", "RESULT"))
    for r in sorted(recs, key=lambda r: r.get("order") or 0):
        name = "%s@%s" % (r["task"]["name"], r["task"]["version"].split(".")[0])
        msgs = [i.get("message", "") for i in (r.get("issues") or [])
                if "end-of-life" in i.get("message", "")]
        nodes = sorted({n for m in msgs for n in re.findall(r"Node version \((\d+)\)", m)})
        print("%-40s %s" % (name, "EOL WARNING (Node %s)" % ",".join(nodes) if msgs else "CLEAN"))
    EOF
  displayName: 'Node EOL warning summary'
  condition: always()
  env:
    SYSTEM_ACCESSTOKEN: $(System.AccessToken)
```
{% endraw %}

```bash

```
