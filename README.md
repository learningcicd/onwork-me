{% raw %}
```yaml

```
{% endraw %}

```bash
      - task: PowerShell@2
        displayName: Check Maven on agent
        condition: succeededOrFailed()
        inputs:
          targetType: inline
          script: |
            $mvn  = Get-Command mvn  -ErrorAction SilentlyContinue
            $java = Get-Command java -ErrorAction SilentlyContinue

            if ($java) { Write-Host "Java found: $($java.Source)"; java -version } else { Write-Host '##vso[task.logissue type=warning]Java not found on agent.' }

            if ($mvn) {
              Write-Host "Maven found: $($mvn.Source)"
              mvn -v
              Write-Host '##vso[task.setvariable variable=mavenAvailable]true'
            } else {
              Write-Host '##vso[task.logissue type=warning]Maven (mvn) not found on agent PATH.'
              Write-Host '##vso[task.setvariable variable=mavenAvailable]false'
            }
```

