{% raw %}
```yaml

# node-eol-task-check.yml
# Runs every task once so you can open each step's log and see whether it has a Node.js warning.
#
# - Each step is called "TASK <name>" and has continueOnError: true, so the run always goes to the end.
# - Inputs and service connections are dummies or left out, so many tasks will FAIL. That's expected.
#   The agent prints the Node warning when it starts the task, before the task's own code runs,
#   so failed steps still show it.
# - Where to look: the run summary page lists every warning with its step name. Each step's log
#   also shows it near the top as "##[warning]Task '...' is dependent on a Node version (..) that is end-of-life".
# - If a task isn't installed in the org, the whole run fails before it starts. Comment that task out.

trigger: none
pr: none

stages:
- stage: NodeWarningCheck
  jobs:
  - job: AgentTasks
    pool:
      vmImage: ubuntu-latest        # or: name: <your-self-hosted-pool>
    timeoutInMinutes: 60
    steps:
    - checkout: none

    - task: CmdLine@2
      displayName: Setup dummy files
      inputs:
        script: |
          mkdir -p $(Build.ArtifactStagingDirectory)/smoke $(Agent.TempDirectory)/cache
          echo smoke > $(Build.ArtifactStagingDirectory)/smoke/readme.txt
          echo smoke > $(Agent.TempDirectory)/cache/x.txt

    - task: AdvancedSecurity-Dependency-Scanning@1
      displayName: TASK AdvancedSecurity-Dependency-Scanning@1
      continueOnError: true
      timeoutInMinutes: 5

    - task: ArchiveFiles@2
      displayName: TASK ArchiveFiles@2
      continueOnError: true
      inputs:
        rootFolderOrFile: $(Build.ArtifactStagingDirectory)/smoke
        archiveFile: $(Agent.TempDirectory)/smoke.zip

    - task: ArtifactoryGenericUpload@2
      displayName: TASK ArtifactoryGenericUpload@2
      continueOnError: true

    - task: ArtifactoryMaven@1
      displayName: TASK ArtifactoryMaven@1
      continueOnError: true

    - task: ArtifactoryPublishBuildInfo@1
      displayName: TASK ArtifactoryPublishBuildInfo@1
      continueOnError: true

    - task: AzureAppConfigurationPush@1
      displayName: TASK AzureAppConfigurationPush@1
      continueOnError: true

    - task: AzureAppServiceSettings@1
      displayName: TASK AzureAppServiceSettings@1
      continueOnError: true

    - task: AzureCLI@2
      displayName: TASK AzureCLI@2
      continueOnError: true

    - task: AzureFunctionApp@2
      displayName: TASK AzureFunctionApp@2
      continueOnError: true

    - task: AzureKeyVault@2
      displayName: TASK AzureKeyVault@2
      continueOnError: true

    - task: AzurePowerShell@5
      displayName: TASK AzurePowerShell@5
      continueOnError: true

    - task: Cache@2
      displayName: TASK Cache@2
      continueOnError: true
      inputs:
        key: 'node-eol-check | "$(Agent.OS)"'
        path: $(Agent.TempDirectory)/cache

    - task: CmdLine@2
      displayName: TASK CmdLine@2
      continueOnError: true
      inputs:
        script: echo smoke

    - task: CopyFiles@2
      displayName: TASK CopyFiles@2
      continueOnError: true
      inputs:
        SourceFolder: $(Build.ArtifactStagingDirectory)/smoke
        TargetFolder: $(Agent.TempDirectory)/copy

    - task: CopyFilesOverSSH@0
      displayName: TASK CopyFilesOverSSH@0
      continueOnError: true

    - task: DotNetCoreCLI@2
      displayName: TASK DotNetCoreCLI@2
      continueOnError: true
      inputs:
        command: custom
        custom: '--info'

    - task: PublishBuildArtifacts@1
      displayName: TASK PublishBuildArtifacts@1
      continueOnError: true
      inputs:
        PathtoPublish: $(Build.ArtifactStagingDirectory)/smoke
        ArtifactName: smoke-build

    - task: DownloadBuildArtifacts@1
      displayName: TASK DownloadBuildArtifacts@1
      continueOnError: true
      inputs:
        buildType: current
        downloadType: single
        artifactName: smoke-build
        downloadPath: $(Agent.TempDirectory)/dl-build

    - task: DownloadPackage@1
      displayName: TASK DownloadPackage@1
      continueOnError: true

    - task: PublishPipelineArtifact@1
      displayName: TASK PublishPipelineArtifact@1
      continueOnError: true
      inputs:
        targetPath: $(Build.ArtifactStagingDirectory)/smoke
        artifact: smoke-pipeline

    - task: DownloadPipelineArtifact@2
      displayName: TASK DownloadPipelineArtifact@2
      continueOnError: true
      inputs:
        buildType: current
        artifactName: smoke-pipeline
        targetPath: $(Agent.TempDirectory)/dl-pipeline

    - task: gulp@0
      displayName: TASK gulp@0
      continueOnError: true

    - task: HelmDeploy@0
      displayName: TASK HelmDeploy@0
      continueOnError: true

    - task: JFrogMaven@1
      displayName: TASK JFrogMaven@1
      continueOnError: true

    - task: JFrogPublishBuildInfo@1
      displayName: TASK JFrogPublishBuildInfo@1
      continueOnError: true

    - task: Kubernetes@1
      displayName: TASK Kubernetes@1
      continueOnError: true

    - task: Maven@4
      displayName: TASK Maven@4
      continueOnError: true

    - task: MicrosoftSecurityDevOps@1
      displayName: TASK MicrosoftSecurityDevOps@1
      continueOnError: true
      timeoutInMinutes: 5

    - task: NodeAndNpmTool@1
      displayName: TASK NodeAndNpmTool@1
      continueOnError: true

    - task: NodeTool@0
      displayName: TASK NodeTool@0
      continueOnError: true
      inputs:
        versionSpec: '20.x'

    - task: Npm@1
      displayName: TASK Npm@1
      continueOnError: true
      inputs:
        command: custom
        customCommand: '--version'

    - task: NuGetToolInstaller@1
      displayName: TASK NuGetToolInstaller@1
      continueOnError: true

    - task: PowerShell@2
      displayName: TASK PowerShell@2
      continueOnError: true
      inputs:
        targetType: inline
        script: Write-Host smoke

    - task: PublishCodeCoverageResults@2
      displayName: TASK PublishCodeCoverageResults@2
      continueOnError: true
      inputs:
        summaryFileLocation: '**/no-such-coverage.xml'

    # Name was cut off in the screenshot. Put in the exact name, then uncomment.
    # - task: PublishHtml@1
    #   displayName: TASK PublishHtml@1
    #   continueOnError: true

    - task: PublishTestResults@2
      displayName: TASK PublishTestResults@2
      continueOnError: true
      inputs:
        testResultsFiles: '**/no-such-results.xml'

    - task: PythonScript@0
      displayName: TASK PythonScript@0
      continueOnError: true
      inputs:
        scriptSource: inline
        script: print("smoke")

    - task: SetValueFromJSON@0
      displayName: TASK SetValueFromJSON@0
      continueOnError: true

    - task: sonar-buildbreaker@8
      displayName: TASK sonar-buildbreaker@8
      continueOnError: true

    - task: SonarQubePublish@8
      displayName: TASK SonarQubePublish@8
      continueOnError: true

    - task: SSH@0
      displayName: TASK SSH@0
      continueOnError: true

    - task: UniversalPackages@0
      displayName: TASK UniversalPackages@0
      continueOnError: true

    - task: UpdateBuildNumber@0
      displayName: TASK UpdateBuildNumber@0
      continueOnError: true

    - task: UseNode@1
      displayName: TASK UseNode@1
      continueOnError: true
      inputs:
        version: '20.x'

    - task: YarnInstaller@3
      displayName: TASK YarnInstaller@3
      continueOnError: true

    - task: Yarn@3
      displayName: TASK Yarn@3
      continueOnError: true

    - task: PostBuildCleanup@3
      displayName: TASK PostBuildCleanup@3
      continueOnError: true

  # ManualValidation only runs in an agentless job and doesn't use Node, so it should never warn.
  # It times out after 1 minute and continues on its own.
  - job: AgentlessTasks
    pool: server
    steps:
    - task: ManualValidation@1
      displayName: TASK ManualValidation@1
      timeoutInMinutes: 1
      inputs:
        notifyUsers: ''
        instructions: Node warning check - no action needed
        onTimeout: resume
```
{% endraw %}

```bash

```
