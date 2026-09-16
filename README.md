{% raw %}
```yaml

```
{% endraw %}

# Physical Partition Over Size Pipeline — Failure Analysis & Fix

## Summary

The `StageMonitoringPhysicalPartitionOverSize` pipeline (production, `p3.yaml`) failed
during the **"Generate Reports"** step with:

```
/mnt/vss/_work/1/s/rxi-deployment-utilities/monitoring/cosmos/physical-partition-over-size.ps1 : Cannot index into a null array.
At /mnt/vss/_work/_temp/azureclitaskscript1789499604265_inlinescript.ps1:32 char:1
+ & "./physical-partition-over-size.ps1" @Params
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : InvalidOperation: (:) [physical-partition-over-size.ps1], RuntimeException
    + FullyQualifiedErrorId : NullArray,physical-partition-over-size.ps1

##[error]Script failed with exit code: 1
```

The failure occurred while the script was analyzing the container
`GRCV_OutboxAnticorruption` — the last "Analize container" line logged before the
error, and it crashed the entire job rather than skipping just that one container.

## Root Cause

The failure is a **PowerShell defect in `physical-partition-over-size.ps1`**, not a
pipeline/YAML or permissions problem.

Inside the per-container loop, the script does this:

```powershell
$res = Invoke-Expression $cmd   # az cosmosdb sql container show ...
Write-Output "Analize container: $($collectionname)"
if (-not [string]::IsNullOrEmpty($res)) {
    $jsonphysinfo = $res | ConvertFrom-Json
    if ($jsonphysinfo[0].resource.statistics.length -gt 0) {   # <-- crashes here
        ...
    }
}
```

The only guard in place checks that the **raw string** `$res` isn't null/empty
*before* parsing it. It never checks that the **parsed object**
(`$jsonphysinfo`) is actually usable afterward. `ConvertFrom-Json` can still
return `$null` — or an object missing `.resource.statistics` — even when `$res`
itself is a non-empty string (e.g. the CLI returned `"null"`, an unexpected
JSON shape, or a response that doesn't contain the `statistics` property for
that particular container).

When that happens, `$jsonphysinfo[0]` evaluates to `$null`, and indexing into
it (`.resource.statistics.length`) throws exactly the error seen in the log:
**"Cannot index into a null array."** Because this is inside the script's
`try {}` block but the `catch {}` only does `Write-Error "Exceptions"` and does
not suppress the underlying terminating error from crashing the whole
`AzureCLI@2` task, the entire "Generate Reports" step — and therefore the
pipeline run — fails, instead of just skipping the one problematic container.

### Why this surfaced now

This exact code path has likely always had the gap; it simply hadn't been hit
by a container returning this shape of response before. The recent move to
production (new Cosmos account, new subscription, new service connection
`RxI-Prod-Leap`) is the most likely trigger for a difference in behavior for
this specific container, but the *pipeline itself* isn't misconfigured — the
script's error handling just isn't resilient to it.

## Fix

Add a proper guard **after** parsing the JSON, before indexing into it, and
`continue` to the next container instead of letting the error propagate and
kill the whole job:

```powershell
if ([string]::IsNullOrEmpty($res)) {
    Write-Warning "Empty response for container $($collectionname) in database $($databasename). Skipping."
    continue
}

$jsonphysinfo = $res | ConvertFrom-Json

if (-not $jsonphysinfo -or
    -not $jsonphysinfo[0] -or
    -not $jsonphysinfo[0].resource -or
    -not $jsonphysinfo[0].resource.statistics) {
    Write-Warning "No partition statistics returned for container $($collectionname) in database $($databasename). Raw response: $res"
    continue
}

if ($jsonphysinfo[0].resource.statistics.length -gt 0) {
    # ...existing processing logic, unchanged...
} else {
    Write-Warning "Container $($collectionname) in database $($databasename) has zero physical partitions reported. Skipping."
    continue
}
```

The full patched loop was provided separately as
`physical-partition-loop-fixed.ps1` — it's a drop-in replacement for the
existing `foreach ($jsoncoll in $jsoncollections) { ... }` block, with the
guard added and everything else unchanged.

### Why this fix is safe

- It mirrors the same "skip on empty/bad response" pattern the script already
  uses one level up, for the `az cosmosdb sql container list` call.
- It doesn't change how any *valid* container response is processed.
- It converts a hard pipeline failure into a per-container skip with a logged
  warning, so the report still completes for every other container.

## Recommended follow-up

- **Keep the `Write-Warning` lines in for the next several runs.** If the
  pipeline consistently flags the same container/database, or the warning
  always follows a specific pattern, that will confirm whether this is a true
  one-off oddity in the API response or something systemic (e.g. always the
  same container type, always right after a certain call, rate-limiting,
  etc.).
- Optionally capture `$res` to the pipeline log or a report file whenever the
  new guard triggers, so the raw API response is available for review without
  having to reproduce the issue.
- Consider adding retry logic (1–2 retries with a short delay) around the
  `az cosmosdb sql container show` call specifically, in case this turns out
  to be a transient/rate-limited response rather than a permanently
  unusual one for that container.

