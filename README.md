{% raw %}
```yaml
condition: or(eq(variables['Build.SourceBranch'], 'refs/heads/master'), eq(variables['Build.Reason'], 'Manual'))
```
{% endraw %}



