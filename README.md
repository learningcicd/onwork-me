```
# templates/build-stage.yml
# Reusable .NET build stage template

parameters:
  - name: artifactName
    type: string
  - name: buildConfiguration
    type: string
    default: 'Release'
  - name: dotnetVersion
    type: string
    default: '8.0.x'
  - name: useGlobalJson
    type: boolean
    default: true
  - name: projectPattern
    type: string
    default: '**/*.csproj'
  - name: stageName
    type: string
    default: 'Build'

stages:
  - stage: ${{ parameters.stageName }}
    displayName: Build
    jobs:
      - job: Build
        displayName: Build Job
        steps:
          - bash: |
              version=$(grep '<Version>' < *.csproj | sed 's/.*<Version>\(.*\)<\/Version>/\1/')
              echo "##vso[task.setvariable variable=VERSION]$version"
              echo "##vso[task.setvariable variable=gitBranch]$(echo $(Build.SourceBranch) | sed -e 's/^refs\/heads\///g' -e 's/\//-/g')"
            displayName: 'Variable Set'

          - task: UseDotNet@2
            displayName: 'Use .NET SDK'
            inputs:
              packageType: 'sdk'
              ${{ if parameters.useGlobalJson }}:
                useGlobalJson: true
              ${{ else }}:
                version: ${{ parameters.dotnetVersion }}

          - task: DotNetCoreCLI@2
            displayName: 'DotNet Restore'
            inputs:
              command: 'restore'
              projects: ${{ parameters.projectPattern }}

          - task: DotNetCoreCLI@2
            displayName: 'Publish build artifact staging directory'
            inputs:
              command: publish
              publishWebProjects: false
              projects: ${{ parameters.projectPattern }}
              arguments: '-c ${{ parameters.buildConfiguration }} --output $(Build.ArtifactStagingDirectory)/$(VERSION)-b$(Build.BuildId)'
              zipAfterPublish: true

          - task: PublishBuildArtifacts@1
            displayName: 'Publish build artifact'
            inputs:
              pathtoPublish: '$(Build.ArtifactStagingDirectory)/$(VERSION)-b$(Build.BuildId)'
              artifactName: ${{ parameters.artifactName }}
```
