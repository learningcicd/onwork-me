{% raw %}
```yaml
# Starter pipeline

# Start with a minimal pipeline that you can customize to build and deploy your code.
# Add steps that build, run tests, deploy, and more:
# https://aka.ms/yaml
trigger: none

parameters:
  - name: APIMInstanceName
    displayName: "APIM Instance Name"
    type: string
    values:
      - rxr-apim-dev-nprod-cus
  - name: Environment
    displayName: "Environment"
    type: string
    values:
      - dev
      - dev2
      - devqe
      - devqe2
  - name: projectName
    displayName: "Project Name"
    values:
      - RxR-SCM
      - RxR-CFS
      - PlatformX-Components
      - RxR-IntegrationServices
    type: string
  - name: repoName
    displayName: "Repository Name"
    type: string
    values:
      - goods-receiving-api
      - interstore-transfer-api
      - inventory-data-factory-api
      - location-api
      - manual-suggested-order-processor
      - order-schedule-processor
      - order-schedule-api
      - product-api
      - product-positioning
      - product-tracking
      - purchase-order-api
      - replenishment-emergency-compute
      - report-manager-api
      - restricted-product-processor
      - return-api
      - return-history
      - routingservice
      - rx-bi-proxy-nodejs
      - stock-api
      - stockcounting-api
      - suggested-order-generator-api
      - supplier-api
      - supplychaincorporatenodebff
      - supplychainstorebff
      - taskmanager
      - rxi-tool-management
      - inventory-getonhandqty
  - name: repoBranch
    displayName: "Branch Name"
    type: string
  - name: targetAPI
    displayName: "Target API"
    type: string
    values:
      - goods-receiving-api
      - interstore-transfer-api-v3
      - inventory-data-factory-api
      - location-api
      - manual-suggested-order-processor
      - nexia
      - order-schedule-processor
      - product-api
      - product-positioning
      - product-tracking
      - purchase-order-api
      - replenishment-emergency-compute
      - report-manager-api
      - restricted-product-processor
      - return-api
      - return-history
      - routingservice
      - rx-bi-proxy
      - stock-api
      - stock-api-v4
      - stock-api-v5
      - stock-enterprise-api
      - stockcounting-api
      - suggested-order-generator-api
      - supplier-api
      - supplychaincorporatenodebff
      - supplychainstorebff
      - taskmanager
      - inventory
  - name: yamldir
    displayName: "Yaml Directory"
    type: string
    default: "/deployment/apim"
appendCommitMessageToRunName: false
pool:
  name: infra-pf-adopl-core-01-centralus-nprod
workspace:
  clean: all
steps:
  - task: PowerShell@2
    inputs:
      targetType: 'inline'
      script: |
        $environment = '${{ parameters.Environment }}'
        $target = '${{ parameters.targetAPI }}'
        Write-Host "Updating $target-$environment"
        $BuildName = "Updating $target-$environment"

        Write-Host "##vso[build.updatebuildnumber]$BuildName"
    displayName: Update Build Name
    condition: succeeded()
  - script: |
      rm -rf "$(Build.BinariesDirectory)/*"
      git clone https://Rx-Renewal:$(SYSTEM.ACCESSTOKEN)@dev.azure.com/Rx-Renewal/"${{parameters['projectName']}}"/_git/"${{parameters['repoName']}}" "$(Build.BinariesDirectory)"
      oldpwd="${PWD}"
      cd "$(Build.BinariesDirectory)"
      git checkout -f "${{parameters['repoBranch']}}"
      cd "${oldpwd}"
    displayName: Checkout Repository ${{parameters['repoName']}}
    condition: succeeded()
  - task: Bash@3
    displayName: "Set variables"
    inputs:
      targetType: "inline"
      script: |
        case "${{parameters['APIMInstanceName']}}" in
          rxr-rxi-nprod-01-cus-apim)
            echo '##vso[task.setvariable variable=envclientsecret]'"$(clientsecret05)"
            ;;
          *)
            echo '##vso[task.setvariable variable=envclientsecret]'"$(clientsecret)"
            ;;
        esac
    condition: succeeded()
  - task: UsePythonVersion@0
    displayName: "Use Python 3.11"
    inputs:
      versionSpec: '3.11'
      addToPath: true
  - task: Bash@3
    displayName: "Verify python3.11"
    inputs:
      targetType: "inline"
      script: |
        if ! command -v python3.11 >/dev/null 2>&1; then
          ln -s "$(command -v python3)" "$(dirname "$(command -v python3)")/python3.11"
        fi
        python3.11 --version
  - task: Bash@3
    displayName: "Create/Update Operations"
    inputs:
      targetType: filePath
      filePath: "$(System.DefaultWorkingDirectory)/api-management/script/updateoperationsprance.sh"
      arguments: "${{parameters['APIMInstanceName']}} ${{parameters['targetAPI']}} ${{parameters['Environment']}} $(envclientsecret) ${{parameters['yamldir']}} ${{parameters['repoName']}}"
    condition: succeeded()
```
{% endraw %}

