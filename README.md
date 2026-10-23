{% raw %}
```yaml

```
{% endraw %}


```sh
for /f "usebackq delims=" %n in (`az keyvault secret list --vault-name rxr-rxi-e2e-02-cus-kv --query "[].name" -o tsv`) do @for /f "usebackq delims=" %v in (`az keyvault secret show --vault-name rxr-rxi-e2e-02-cus-kv --name %n --query value -o tsv 2^>nul`) do @if /i "%v"=="163a83ee-c063-460b-a061-4f36b4b96566" echo MATCH: %n
```
