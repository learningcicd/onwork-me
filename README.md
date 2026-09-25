{% raw %}
```yaml
#!/bin/bash
set +x
apimname=$1 # {parameters['APIMInstanceName']}
targetapi=$2
environment=$3 # {parameters['Environment']}
envclientsecret=$4
yamldir=$5
reponame=$6
# policies=$7

python3 --version
python3 -m pip --version

python3 -m pip install --upgrade setuptools -q
python3 -m pip install argcomplete -q
python3 -m pip install keyring -q
python3 -m pip install swagger_spec_validator -q
python3 -m pip install openapi-spec-validator -q
python3 -m pip install html_reports -q
python3 -m pip install click -q
python3 -m pip install jq -q
# Install requests and prance together so pip picks a prance version compatible with requests 2.25
python3 -m pip install "requests==2.25.0" prance -q

export PATH=$PATH:"$HOME/.local/bin"

prance --version
case "${environment}" in
  dev)
    policies=$(echo "policy_cors_rxr-apim-dev-leap.xml")
    ;;
  devqe)
    policies=$(echo "policy_cors_rxr-apim-devqe-leap.xml")
    ;;
  dev2)
    policies=$(echo "policy_cors_rxr-apim-dev2-leap.xml")
    ;;
  devqe2)
    policies=$(echo "policy_cors_rxr-apim-devqe2-leap.xml")
    ;;
  e2e)
    policies=$(echo "policy_cors_rxr-apim-e2e-leap.xml")
    ;;
  e2e02)
    policies=$(echo "policy_cors_rxr-apim-e2e-02-leap.xml")
    ;;
  uat)
    policies=$(echo "policy_cors_rxr-apim-uat-leap.xml")
    ;;
  pet-01)
    policies=$(echo "policy_cors_rxr-apim-pet-01.xml")
    ;;
  perf)
    policies=$(echo "policy_cors_rxr-apim-perf-01.xml")
    ;;
  perf-02)
    policies=$(echo "policy_cors_rxr-apim-perf2-leap.xml")
    ;;
  prodfix05)
    policies=$(echo "policy_cors_rxr-apim-prodfix05-leap.xml")
    ;;
  prodfix)
    policies=$(echo "policy_cors_rxr-apim-prodfix05-leap.xml")
    ;;
  *)
    policies='none'
esac
case "${apimname}" in
  rxr-rxi-nprod-01-cus-apim)
    envsubscription=$(echo "${SUBSCRIPTIONID05}")
    envclientid=$(echo "${CLIENTID05}")
    resourcegroup=$(echo "${RGLEAP05}")
    ;;
  *)
    envsubscription=$(echo "${SUBSCRIPTIONIDLEAPENV}")
    envclientid=$(echo "${CLIENTID}")
    resourcegroup=$(echo "${RGLEAP}")
    ;;
esac
if [[ "${policies}" == '' || "${policies}" == 'none' ]]; then
    echo "WARNING: path to policy files is empty, setting to default per environment"
    case "${apimname}" in
    rxr-apim-dev-nprod-cus | rxr-rxi-nprod-01-cus-apim)
        policy=$(echo "-p ./api-management/apim/default_policy_rxr-apim-leap.xml")
        ;;
    *)
        echo "Unknown APIM Instance. This condition should never be met in normal program workflow. Exiting."
        ;;
    esac
else
    echo "Setting policy file to ${policies}"
    policy=$(echo "-p ./api-management/apim/${policies}")
fi
if [[ "${apimname}" != '' ]]; then
    apiparameters=$(echo "--subscription-id ${envsubscription} --tenant-id ${TENANTID} -s ${apimname} -g ${resourcegroup}")
    echo "apiparameters: ${apiparameters}"
    params=$(echo " ${apiparameters} --version ${APIVERSION} --client-id ${envclientid} --client-secret ${envclientsecret} ${policy}")
    echo "params: ${params}"
fi

if [[ "${targetapi}" != '' ]]; then
    targetapi=$(echo "${targetapi}"-"${environment}")
else
    targetapi=$(echo "${reponame}"-"${environment}")
fi

apiparams=$(echo "-n ${targetapi} ${params}")
echo "apiparams: ${apiparams}"
env | grep -i api
echo "Create Update Operations"
list=$(find "${BUILD_BINARIESDIRECTORY}"/${yamldir} -type f \( ! -iname "*CommonSchemas*.yaml" -and ! -iname "*Zalando*.yaml" \))
for file in $list; do
    echo $file
    python3 -u './api-management/script/apiCtlAzureImprovementPrance.py' operation update -y $file ${apiparams} --verbose
done
```
{% endraw %}

