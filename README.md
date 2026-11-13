{% raw %}
```yaml
trigger: none
pr: none

parameters:
- name: nodeVersion
  displayName: 'Max Node version visible to the agent for this run'
  type: string
  default: '24'
  values:
  - '24'
  - '20'
  - '16'
  - '10'

pool:
  name: 'agent-for-node-runner-test'   # <-- your dedicated single agent

variables:
  system.debug: true
  externalsDir: '$(Agent.HomeDirectory)/externals'
  hiddenDir: '$(Agent.HomeDirectory)/externals/_node_test_hidden'

steps:
- bash: |
    set -uo pipefail
    mkdir -p "$(hiddenDir)"
    if [ -n "$(ls -A "$(hiddenDir)" 2>/dev/null)" ]; then
      echo "##vso[task.logissue type=error]$(hiddenDir) is not empty. A previous run did not restore cleanly - fix by hand before continuing (move its contents back into externals/, then remove it)."
      exit 1
    fi

    # Folders ranked above the requested cap, newest first.
    case "${{ parameters.nodeVersion }}" in
      24) ABOVE="" ;;
      20) ABOVE="node24" ;;
      16) ABOVE="node24 node20_1" ;;
      10) ABOVE="node24 node20_1 node16" ;;
    esac

    # Pre-flight: every folder we'd need to hide must actually be movable by
    # this account before we touch anything. Moving a directory needs write
    # permission on it (to remove entries while relocating/deleting it), not
    # just on its parent - so check that explicitly rather than finding out
    # mid-move and leaving the agent half-hidden.
    BLOCKED=""
    for d in $ABOVE; do
      p="$(externalsDir)/$d"
      if [ -d "$p" ] && [ ! -w "$p" ]; then
        BLOCKED="$BLOCKED $d"
        echo "##[error]Not writable by $(whoami): $p"
        ls -ld "$p"
      fi
    done
    if [ -n "$BLOCKED" ]; then
      echo "##vso[task.logissue type=error]Aborting before hiding anything - this account can't move:$BLOCKED. Fix ownership/permissions on the agent (so the account running this job can write to these folders) and retry."
      exit 1
    fi

    for d in $ABOVE; do
      if [ -d "$(externalsDir)/$d" ]; then
        mv "$(externalsDir)/$d" "$(hiddenDir)/$d"
        echo "Hid $d"
      fi
    done

    echo "Externals now visible to the agent:"
    ls -1 "$(externalsDir)" | grep -i '^node' || echo "  (none)"
  displayName: 'Cap agent at Node ${{ parameters.nodeVersion }} (hide newer runners)'

- checkout: self
  displayName: 'Checkout@1'

- bash: |
    echo "Capped at Node ${{ parameters.nodeVersion }}. Runners now visible:"
    ls -1 "$(externalsDir)" | grep -i '^node' || true
  displayName: 'Node runtime visibility for this run'

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

# Print one line per task: which Node runtime ran it, EOL warning, or no
# compatible handler found at all.
- bash: |
    curl -s -H "Authorization: Bearer $SYSTEM_ACCESSTOKEN" \
      "$(System.CollectionUri)$(System.TeamProject)/_apis/build/builds/$(Build.BuildId)/timeline?api-version=7.1" \
      -o timeline.json
    python3 - <<'EOF'
    import json, re
    recs = [r for r in json.load(open("timeline.json"))["records"]
            if r["type"] == "Task" and r.get("task")]
    print("Node cap for this run: ${{ parameters.nodeVersion }}\n")
    print("%-40s %-10s %s" % ("TASK", "RESULT", "DETAIL"))
    for r in sorted(recs, key=lambda r: r.get("order") or 0):
        name = "%s@%s" % (r["task"]["name"], r["task"]["version"].split(".")[0])
        issues = r.get("issues") or []
        warn = [i.get("message", "") for i in issues
                if "end-of-life" in i.get("message", "")]
        errs = [i.get("message", "") for i in issues if i.get("type") == "error"]
        if warn:
            nodes = sorted({n for m in warn for n in re.findall(r"Node version \((\d+)\)", m)})
            print("%-40s %-10s Node %s" % (name, "EOL WARN", ",".join(nodes)))
        elif r.get("result") in ("failed", "canceled") and errs:
            print("%-40s %-10s %s" % (name, "FAILED", errs[0][:90]))
        else:
            print("%-40s %-10s -" % (name, "CLEAN"))
    EOF
  displayName: 'Node handler summary for this cap'
  condition: always()
  env:
    SYSTEM_ACCESSTOKEN: $(System.AccessToken)

# Always restore, even if the job was cancelled or a step above failed hard.
# Puts the agent back to its default: node24 (and every other runner) visible.
- bash: |
    set -uo pipefail
    shopt -s nullglob
    moved=0
    for d in "$(hiddenDir)"/*/; do
      name="$(basename "$d")"
      mv "$d" "$(externalsDir)/$name"
      echo "Restored $name"
      moved=1
    done
    [ "$moved" = "1" ] || echo "Nothing to restore."
    rmdir "$(hiddenDir)" 2>/dev/null || true

    echo "Externals now visible to the agent:"
    ls -1 "$(externalsDir)" | grep -i '^node' || true

    if [ ! -d "$(externalsDir)/node24" ]; then
      echo "##vso[task.logissue type=error]node24 is NOT present in $(externalsDir) after restore. The agent is NOT back to its Node 24 default - check $(hiddenDir) by hand before letting anything else run on this agent."
      exit 1
    fi
    echo "Confirmed: node24 is back in place - agent is at its default."
  displayName: 'Restore Node runners on agent (back to Node 24 default)'
  condition: always()
```
{% endraw %}

```bash

```
