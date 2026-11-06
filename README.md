{% raw %}
```yaml
trigger: none

parameters:
  - name: agentPool
    displayName: Agent pool
    type: string
    default: rxi-utils-pool
  - name: appConfigEndpoint
    displayName: App Configuration endpoint (https://<store>.azconfig.io)
    type: string

jobs:
  - job: TaskRunnerSmokeTest
    displayName: Verify marketplace tasks
    variables:
      AGENT_RESTRICT_EOL_NODE_VERSIONS: 'false'
      smokeBuildName: task-runner-smoke-test
      smokeDir: $(Agent.TempDirectory)/task-runner-smoke-test
    pool:
      name: ${{ parameters.agentPool }}
    steps:
      - task: NodeAndNpmTool@1
        displayName: Test NodeAndNpmTool Handler
        inputs:
          versionSpec: '14.x' # 14.x for ubuntu 18

      - task: PowerShell@2
        displayName: Verify Node and npm executables
        inputs:
          targetType: inline
          script: |
            node --version
            if ($LASTEXITCODE -ne 0) { throw 'Node verification failed.' }
            npm --version
            if ($LASTEXITCODE -ne 0) { throw 'npm verification failed.' }

      - task: YodLabs.VariableTasks.SetVariableFromJSON.SetValueFromJSON@0
        displayName: Test SetValueFromJSON Handler
        condition: succeededOrFailed()
        inputs:
          variableName: taskRunnerJsonResult
          jsonPathExpression: '$.result'
          json: '{"result":"task-runner-ok"}'

      - task: PowerShell@2
        displayName: Verify extracted JSON value
        condition: succeededOrFailed()
        inputs:
          targetType: inline
          script: |
            if ($env:JSON_RESULT -ne 'task-runner-ok') {
              throw 'SetValueFromJSON did not produce the expected value.'
            }
            Write-Host 'JSON extraction verified.'
        env:
          JSON_RESULT: $(taskRunnerJsonResult)

      # Creates every input file the later tasks need, so no step depends on repo contents
      - task: PowerShell@2
        displayName: Create disposable test files
        condition: succeededOrFailed()
        inputs:
          targetType: inline
          script: |
            $dir = $env:SMOKE_DIR
            New-Item -ItemType Directory -Force -Path $dir | Out-Null

            Set-Content -Path (Join-Path $dir 'smoke-test.txt') -Value "Task runner smoke test: $env:BUILD_BUILDID"

            Set-Content -Path (Join-Path $dir 'appsettings.json') -Value '{ "TaskRunnerSmokeTest": { "Result": "ok" } }'

            @'
            <project xmlns="http://maven.apache.org/POM/4.0.0">
              <modelVersion>4.0.0</modelVersion>
              <groupId>com.rxi.smoketest</groupId>
              <artifactId>task-runner-smoke-test</artifactId>
              <version>1.0.0</version>
              <packaging>pom</packaging>
            </project>
            '@ | Set-Content -Path (Join-Path $dir 'pom.xml')

            Get-ChildItem $dir
        env:
          SMOKE_DIR: $(smokeDir)

      - task: JFrog.jfrog-artifactory-vsts-extension.artifactory-generic-upload.ArtifactoryGenericUpload@1
        displayName: Test ArtifactoryGenericUpload Handler
        condition: succeededOrFailed()
        inputs:
          artifactoryService: jfrog
          specSource: taskConfiguration
          fileSpec: |
            {
              "files": [
                {
                  "pattern": "$(smokeDir)/smoke-test.txt",
                  "target": "random/task-runner-smoke-test/$(Build.BuildId)/",
                  "flat": "true"
                }
              ]
            }
          failNoOp: true
          collectBuildInfo: true
          buildName: $(smokeBuildName)
          buildNumber: $(Build.BuildId)

      - task: ArtifactoryPublishBuildInfo@1
        displayName: Test ArtifactoryPublishBuildInfo handler
        condition: succeededOrFailed()
        inputs:
          artifactoryService: jfrog
          buildName: $(smokeBuildName)
          buildNumber: $(Build.BuildId)

      - task: AzureAppConfigurationPush@2
        displayName: Test AzureAppConfigurationPush handler
        condition: succeededOrFailed()
        inputs:
          azureSubscription: RxI-NProd-05
          AppConfigurationEndpoint: ${{ parameters.appConfigEndpoint }}
          ConfigurationFile: '$(smokeDir)/appsettings.json'

      - task: Maven@2
        displayName: Test Maven Node handler
        condition: succeededOrFailed()
        inputs:
          mavenPomFile: '$(smokeDir)/pom.xml'
          goals: 'validate'
          publishJUnitResults: false

      - task: PostBuildCleanup@4
        displayName: Test PostBuildCleanup (agent workspace cleanup)
        condition: always()
```
{% endraw %}

```bash

```

