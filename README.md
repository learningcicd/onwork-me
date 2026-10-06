{% raw %}
```yaml
# APIM Connectivity Test - runs a single curl against the APIM endpoint

trigger: none
pr: none

parameters:
  - name: agentPool
    type: string
    default: '<AGENT_POOL_NAME>'
  - name: apimUrl
    type: string
    default: 'https://rxiapi.walgreens.com/v1/sc/prod05/apps-manager'

pool:
  name: ${{ parameters.agentPool }}

steps:
  - checkout: none

  - bash: |
      curl -k -sS --max-time 30 -w '\nHTTP status: %{http_code}\n' "${{ parameters.apimUrl }}"
    displayName: curl APIM endpoint
```
{% endraw %}

