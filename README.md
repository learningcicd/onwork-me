{% raw %}
```yaml

```
{% endraw %}

```bash
  <#
.SYNOPSIS
  Lists every pipeline in one or more Azure DevOps projects and the agent pool(s) each uses,
  and saves the result as an Excel workbook.

.DESCRIPTION
  Workbook sheets:
    Pipelines     - one row per pipeline definition (all folders, including never-run/disabled ones)
    Pool Summary  - one row per pool: how many pipelines use it, and which
    UniqueTaskRunners - every distinct value from the TaskRunners column (e.g. Bash@3.279.0=Node 20), one per row

  Pipelines columns:
    Project, Type, Pipeline
    PipelineUrl     - full link to the pipeline in Azure DevOps (clickable)
    Folder
    LatestRunPools  - pools used by the most recent run (what the pipeline uses today)
    LatestRun       - that run's number/name
    LatestRunDate   - when that run's last job finished (local time, yyyy-MM-dd HH:mm).
                      If the run is older than the pool job history, it comes from the
                      definition's latest build / last release instead, and LatestRunPools
                      shows '(not in pool job history)'. '(no runs kept)' = no run within retention.
    LatestRunBy     - who started that run (for CI/schedule runs: the person the run was queued for)
    LatestRunReason - why it ran: manual, individualCI, batchedCI, schedule, pullRequest, ...
    NodeWarnings    - only with -NodeWarnings: number of Node.js-related warnings in the latest run
                      (0 = checked, none found; blank = not checked: no run, or older than -Since)
    NodeWarningTasks- the tasks behind those warnings, e.g. AzureCLI@2 (Node 10)
    LowestNodeRunner- only with -NodeWarnings: oldest Node.js runner any task in the latest run uses
                      (Node 6/10/16 = breaks or changes behaviour after Nov 24 2026; Node 20/24 = OK)
    TaskRunners     - every task in the latest run with the runner it uses, oldest first,
                      e.g. AzureCLI@1.0.1=Node 10; CopyFiles@2.246.0=Node 20; PowerShell@2.247.1=Node 20
    LatestRunAgents - agent(s) that ran the latest run
    PoolsUsed       - every pool any run used in the job-history window, most recent first
    PoolLastUsed    - same pools with the date each was last used
    ConfiguredPools - pools set on the definition (classic build phases, classic release stages)
    DefaultPool     - YAML only: the default queue on the pipeline settings (the YAML 'pool:' overrides it)
    Source          - YAML only: repo and YAML file path

  Excel output uses the ImportExcel module if installed, otherwise Excel itself (must be installed).
  If neither is available it falls back to a .csv with the same columns.

.EXAMPLE
  .\ado-node-runner-report.ps1 -Project "RxI-Integration"
  .\ado-node-runner-report.ps1 -Project "RxI-DevOps-Deployment","RxI-Integration","RxR-SCM"
  .\ado-node-runner-report.ps1 -Project "RxR-SCM" -NodeWarnings -Since 2026-07-01
  .\ado-node-runner-report.ps1 -Project "RxR-SCM" -OutFile .\scm-pools.xlsx -JobHistory 10000
#>
param(
  [Parameter(Mandatory)][string[]]$Project,
  [string]$Org = 'Rx-Renewal',
  [int]$JobHistory = 5000,
  [string]$OutFile,
  [switch]$NodeWarnings,     # kept for compatibility: the Node.js check is now ON by default
  [switch]$SkipNodeCheck,    # turn the Node.js warning / runner check off (faster)
  [string]$Since = ''        # only check runs on/after this date, e.g. 2026-07-01 (blank = all runs)
)
$ErrorActionPreference = 'Stop'
$NodeWarnings = -not $SkipNodeCheck
if ($NodeWarnings) {
  $sinceMsg = if ($Since) { "latest runs since $Since" } else { 'all latest runs (use -Since 2026-07-01 to limit)' }
  Write-Host "Node.js check: ON - $sinceMsg" -ForegroundColor Cyan
} else {
  Write-Host "Node.js check: OFF (-SkipNodeCheck)" -ForegroundColor Yellow
}
if (-not $OutFile) {
  $tag = if ($Project.Count -eq 1) { $Project[0] -replace '[^\w-]', '_' } else { "$($Project.Count)-projects" }
  $OutFile = ".\pipeline-pools-$tag.xlsx"
}

# Azure DevOps token from the az login session (no PAT needed)
$token = az account get-access-token --resource 499b84ac-1321-427f-aa17-267ca6975798 --query accessToken -o tsv
if (-not $token) { throw "No token. Run 'az login' first." }
$H    = @{ Authorization = "Bearer $token" }
$base = "https://dev.azure.com/$Org"
$vsrm = "https://vsrm.dev.azure.com/$Org"

function Get-AdoAll([string]$Url) {
  $all = [System.Collections.Generic.List[object]]::new()
  $ct  = $null
  do {
    $u = if ($ct) { "$Url&continuationToken=$([uri]::EscapeDataString($ct))" } else { $Url }
    $resp = Invoke-RestMethod -Uri $u -Headers $H -ResponseHeadersVariable rh
    if ($resp.value) { $all.AddRange([object[]]$resp.value) }
    $k  = $rh.Keys | Where-Object { $_ -ieq 'x-ms-continuationtoken' } | Select-Object -First 1
    $ct = if ($k) { @($rh[$k])[0] } else { $null }
  } while ($ct)
  $all
}

# Leading comma: stops PowerShell from unrolling the empty set into $null on return
function New-Set { , [System.Collections.Generic.HashSet[string]]::new([StringComparer]::OrdinalIgnoreCase) }
function Join-Set($s) { if ($s -and $s.Count) { ($s | Sort-Object) -join '; ' } else { '' } }

# Observed-pool helpers: most recently used pool first
function Format-Used($h) {
  if (-not $h -or $h.Count -eq 0) { return '' }
  ($h.GetEnumerator() | Sort-Object Value -Descending | ForEach-Object Key) -join '; '
}
function Format-UsedDates($h) {
  if (-not $h -or $h.Count -eq 0) { return '' }
  ($h.GetEnumerator() | Sort-Object Value -Descending |
    ForEach-Object { "$($_.Key): $($_.Value.ToString('yyyy-MM-dd'))" }) -join '; '
}
# Turn Node warning messages into a short list like "AzureCLI@2 (Node 10)"
function Get-NodeSummary($msgs) {
  $items = foreach ($m in @($msgs)) {
    $m = [string]$m
    if ($m -match "Task '(?<t>[^']+)' version (?<v>[\d.]+).*?Node(?: version)?\s*\(?(?<n>\d+)") {
      "$($Matches.t)@$($Matches.v) (Node $($Matches.n))"
    } elseif ($m -match 'Node(?:\.js)?\s*(?<n>\d+)') {
      "Node $($Matches.n): " + $m.Substring(0, [Math]::Min(90, $m.Length))
    } else {
      $m.Substring(0, [Math]::Min(100, $m.Length))
    }
  }
  @($items | Sort-Object -Unique) -join '; '
}

# Node.js runner helpers: execution handler name -> Node version
function Get-HandlerNode([string]$h) {
  switch -Regex ($h) {
    '^Node$'   { return 6 }
    '^Node10'  { return 10 }
    '^Node16'  { return 16 }
    '^Node20'  { return 20 }
    '^Node24'  { return 24 }
    default    { return $null }
  }
}
# The agent runs a task on the newest Node handler the task declares
function Get-BestRunner($handlers) {
  $hs = @($handlers | Where-Object { $_ })
  if ($hs.Count -eq 0) { return [pscustomobject]@{ Node = $null; Label = 'unknown' } }
  $nodes = @($hs | ForEach-Object { Get-HandlerNode $_ } | Where-Object { $_ })
  if ($nodes.Count) {
    $n = ($nodes | Measure-Object -Maximum).Maximum
    return [pscustomobject]@{ Node = $n; Label = "Node $n" }
  }
  [pscustomobject]@{ Node = $null; Label = $hs[0] }   # e.g. PowerShell3, Process
}
# Load one task's definitions (all versions) the first time we see it, then cache
function Import-TaskDefinition($id) {
  $id = "$id".ToLower()
  if ($taskLoaded.ContainsKey($id)) { return }
  $taskLoaded[$id] = $true
  try {
    $defs = (Invoke-RestMethod -Uri "$base/_apis/distributedtask/tasks/$($id)?allVersions=true&api-version=7.1" -Headers $H).value
    foreach ($t in @($defs)) {
      $hs = @()
      foreach ($sec in 'execution', 'prejobexecution', 'postjobexecution') {
        if ($t.$sec) { $hs += @($t.$sec.PSObject.Properties.Name) }
      }
      $hs = @($hs | Sort-Object -Unique)
      $vs = "$($t.version.major).$($t.version.minor).$($t.version.patch)"
      $taskCat["$id|$vs"] = $hs
      $mk = "$id|$($t.version.major)"
      $rank = [long]$t.version.minor * 100000 + [long]$t.version.patch
      if (-not $taskCatMajor.ContainsKey($mk) -or $rank -gt $taskCatMajor[$mk].Rank) {
        $taskCatMajor[$mk] = @{ Rank = $rank; Handlers = $hs }
      }
    }
    if (-not @($defs).Count) { $script:taskMissing++ }
  } catch {
    $script:taskMissing++
    if (-not $script:taskErrShown) { Write-Warning "Task definition not read ($id): $($_.Exception.Message)"; $script:taskErrShown = $true }
  }
}
function Get-TaskHandlers($id, [string]$ver) {
  Import-TaskDefinition $id
  $id = "$id".ToLower()
  $k = "$id|$ver"
  if ($taskCat.ContainsKey($k) -and @($taskCat[$k]).Count) { return $taskCat[$k] }
  $maj = ($ver -split '\.')[0]
  $k2 = "$id|$maj"
  if ($taskCatMajor.ContainsKey($k2)) { return $taskCatMajor[$k2].Handlers }
  $null
}

function Get-RunCols($key) {
  $lr = $latest[$key]
  [pscustomobject]@{
    PoolsUsed      = Format-Used $observed[$key]
    PoolLastUsed   = Format-UsedDates $observed[$key]
    LatestRun      = if ($lr) { $lr.Name } else { '' }
    LatestRunDate  = if ($lr -and $lr.Date) { $lr.Date.ToLocalTime().ToString('yyyy-MM-dd HH:mm') } else { '' }
    LatestRunPools = if ($lr) { Join-Set $lr.Pools } else { '' }
    LatestRunId    = if ($lr) { $lr.Id } else { $null }
  }
}

# --- Task definitions cache (which Node runner each task version declares); loaded per task on demand ---
$taskCat = @{}; $taskCatMajor = @{}; $taskLoaded = @{}
$script:taskMissing = 0; $script:taskErrShown = $false

# --- Resolve projects ---
$projInfo = @{}   # projectId -> name
foreach ($name in $Project) {
  try {
    $pr = Invoke-RestMethod -Uri "$base/_apis/projects/$([uri]::EscapeDataString($name))?api-version=7.1" -Headers $H
    $projInfo[$pr.id] = $pr.name
    Write-Host "Project found: $($pr.name)"
  } catch {
    Write-Warning "Skipping project '$name': it doesn't exist under this name, or you don't have access to it."
  }
}
if ($projInfo.Count -eq 0) { throw "None of the projects could be read. Check the names with: az devops project list --org $base --query `"value[].name`" -o tsv" }

# --- Observed: which pools each pipeline's jobs ran on (read once for all projects) ---
Write-Host "Reading job history from every pool in $Org (this can take a minute)..."
$observed = @{}   # "projectId|planType|definitionId" -> @{ poolName = last used time }
$latest   = @{}   # "projectId|planType|definitionId" -> @{ Id; Name; Pools } for the most recent run
$oldest   = @{}   # projectId -> oldest finishTime seen
$pools = (Invoke-RestMethod -Uri "$base/_apis/distributedtask/pools?api-version=7.1" -Headers $H).value
foreach ($pool in $pools) {
  try {
    $jobs = (Invoke-RestMethod -Uri "$base/_apis/distributedtask/pools/$($pool.id)/jobrequests?completedRequestCount=$JobHistory&api-version=7.1-preview.1" -Headers $H).value
  } catch {
    Write-Warning "Skipped pool '$($pool.name)': $($_.Exception.Message)"
    continue
  }
  foreach ($j in $jobs) {
    if (-not $projInfo.ContainsKey([string]$j.scopeId) -or -not $j.definition) { continue }
    $key = "$($j.scopeId)|$($j.planType)|$($j.definition.id)"
    $t = if ($j.finishTime) { [datetime]$j.finishTime } elseif ($j.assignTime) { [datetime]$j.assignTime } else { [datetime]$j.queueTime }

    # pool -> most recent time a job of this pipeline ran on it
    if (-not $observed.ContainsKey($key)) { $observed[$key] = @{} }
    if (-not $observed[$key].ContainsKey($pool.name) -or $t -gt $observed[$key][$pool.name]) { $observed[$key][$pool.name] = $t }

    # latest run (highest run id) and the pools its jobs used
    $runId = [long]$j.owner.id
    if (-not $latest.ContainsKey($key) -or $runId -gt $latest[$key].Id) {
      $latest[$key] = @{ Id = $runId; Name = [string]$j.owner.name; Pools = (New-Set); Date = $t }
    }
    if ($runId -eq $latest[$key].Id) {
      [void]$latest[$key].Pools.Add($pool.name)
      if ($t -gt $latest[$key].Date) { $latest[$key].Date = $t }   # last job of that run
    }

    if ($j.finishTime) {
      $ft = [datetime]$j.finishTime
      if (-not $oldest[[string]$j.scopeId] -or $ft -lt $oldest[[string]$j.scopeId]) { $oldest[[string]$j.scopeId] = $ft }
    }
  }
}

$rows = [System.Collections.Generic.List[object]]::new()

foreach ($projId in $projInfo.Keys) {
  $projName = $projInfo[$projId]
  $p = [uri]::EscapeDataString($projName)
  Write-Host "Project $projName ..."
  $buildFill   = [System.Collections.Generic.List[object]]::new()
  $releaseFill = [System.Collections.Generic.List[object]]::new()

  # queue id -> pool name (queues are per project)
  $queues = @{}
  (Invoke-RestMethod -Uri "$base/$($p)/_apis/distributedtask/queues?api-version=7.1-preview.1" -Headers $H).value |
    ForEach-Object { $queues[[string]$_.id] = $_.pool.name }
  $poolName = {
    param($queueId)
    if (-not $queueId) { return $null }
    $n = $queues[[string]$queueId]
    if ($n) { $n } else { "queue#$queueId" }
  }

  # Build pipelines (YAML + classic), all folders
  $defs = Get-AdoAll "$base/$($p)/_apis/build/definitions?includeAllProperties=true&includeLatestBuilds=true&queryOrder=definitionNameAscending&api-version=7.1"
  foreach ($d in $defs) {
    $isYaml = $d.process.type -eq 2
    $cfg = New-Set
    $defaultPool = & $poolName $d.queue.id
    if (-not $isYaml) {
      if ($defaultPool) { [void]$cfg.Add($defaultPool) }
      foreach ($ph in @($d.process.phases)) {
        $n = & $poolName $ph.target.queue.id
        if ($n) { [void]$cfg.Add($n) }
      }
    }
    $rc = Get-RunCols "$projId|Build|$($d.id)"
    if (-not $rc.LatestRunDate) {
      # Not in pool job history: fall back to the definition's latest build (goes back to build retention)
      $lb = if ($d.latestBuild) { $d.latestBuild } else { $d.latestCompletedBuild }
      if ($lb) {
        $dt = if ($lb.finishTime) { $lb.finishTime } elseif ($lb.startTime) { $lb.startTime } else { $lb.queueTime }
        $rc.LatestRunId    = $lb.id
        $rc.LatestRun      = [string]$lb.buildNumber
        $rc.LatestRunDate  = if ($dt) { ([datetime]$dt).ToLocalTime().ToString('yyyy-MM-dd HH:mm') } else { '' }
        $rc.LatestRunPools = '(not in pool job history)'
      } else {
        $rc.LatestRun = '(no runs kept)'
      }
    }
    $row = [pscustomobject]@{
      Project         = $projName
      Type            = if ($isYaml) { 'YAML' } else { 'Classic build' }
      Pipeline        = $d.name
      PipelineUrl     = "$base/$($p)/_build?definitionId=$($d.id)"
      Folder          = $d.path
      LatestRunPools  = $rc.LatestRunPools
      LatestRun       = $rc.LatestRun
      LatestRunDate   = $rc.LatestRunDate
      LatestRunBy     = ''
      LatestRunReason = ''
      NodeWarnings    = ''
      NodeWarningTasks = ''
      LowestNodeRunner = ''
      TaskRunners     = ''
      LatestRunAgents = ''
      PoolsUsed       = $rc.PoolsUsed
      PoolLastUsed    = $rc.PoolLastUsed
      ConfiguredPools = Join-Set $cfg
      DefaultPool     = if ($isYaml) { $defaultPool } else { '' }
      Source          = if ($isYaml) { "$($d.repository.name):$($d.process.yamlFilename)" } else { '' }
    }
    $rows.Add($row)
    if ($rc.LatestRunId) { $buildFill.Add(@{ Row = $row; Id = $rc.LatestRunId }) }
  }

  # Classic release pipelines
  try {
    $rdefs = Get-AdoAll "$vsrm/$($p)/_apis/release/definitions?`$expand=lastRelease&api-version=7.1"
  } catch {
    Write-Warning "Release definitions not read for ${projName}: $($_.Exception.Message)"
    $rdefs = @()
  }
  foreach ($rd in $rdefs) {
    $full = Invoke-RestMethod -Uri "$vsrm/$($p)/_apis/release/definitions/$($rd.id)?api-version=7.1" -Headers $H
    $cfg = New-Set
    foreach ($envr in @($full.environments)) {
      foreach ($ph in @($envr.deployPhases)) {
        $qid = $ph.deploymentInput.queueId
        if ($qid -gt 0) { [void]$cfg.Add((& $poolName $qid)) }
        elseif ($ph.phaseType -eq 'machineGroupBasedDeployment') { [void]$cfg.Add('(deployment group)') }
      }
    }
    $rc = Get-RunCols "$projId|Release|$($rd.id)"
    if (-not $rc.LatestRunDate) {
      # Not in pool job history: fall back to the definition's last release
      if ($rd.lastRelease) {
        $rc.LatestRunId    = $rd.lastRelease.id
        $rc.LatestRun      = [string]$rd.lastRelease.name
        $rc.LatestRunDate  = if ($rd.lastRelease.createdOn) { ([datetime]$rd.lastRelease.createdOn).ToLocalTime().ToString('yyyy-MM-dd HH:mm') } else { '' }
        $rc.LatestRunPools = '(not in pool job history)'
      } else {
        $rc.LatestRun = '(no runs kept)'
      }
    }
    $row = [pscustomobject]@{
      Project         = $projName
      Type            = 'Classic release'
      Pipeline        = $full.name
      PipelineUrl     = "$base/$($p)/_release?_a=releases&view=all&definitionId=$($rd.id)"
      Folder          = $full.path
      LatestRunPools  = $rc.LatestRunPools
      LatestRun       = $rc.LatestRun
      LatestRunDate   = $rc.LatestRunDate
      LatestRunBy     = ''
      LatestRunReason = ''
      NodeWarnings    = ''
      NodeWarningTasks = ''
      LowestNodeRunner = ''
      TaskRunners     = ''
      LatestRunAgents = ''
      PoolsUsed       = $rc.PoolsUsed
      PoolLastUsed    = $rc.PoolLastUsed
      ConfiguredPools = Join-Set $cfg
      DefaultPool     = ''
      Source          = ''
    }
    $rows.Add($row)
    if ($rc.LatestRunId) { $releaseFill.Add(@{ Row = $row; Id = $rc.LatestRunId }) }
  }

  # --- Who started the latest run (looked up in batches of 50 runs) ---
  Write-Host "  Looking up who ran the latest runs..."
  $info = @{}
  $ids = @($buildFill | ForEach-Object { [string]$_.Id } | Sort-Object -Unique)
  for ($i = 0; $i -lt $ids.Count; $i += 50) {
    $chunk = $ids[$i..([Math]::Min($i + 49, $ids.Count - 1))] -join ','
    try {
      (Invoke-RestMethod -Uri "$base/$($p)/_apis/build/builds?buildIds=$chunk&api-version=7.1" -Headers $H).value |
        ForEach-Object { $info["b$($_.id)"] = @{ By = $_.requestedFor.displayName; Reason = [string]$_.reason } }
    } catch { Write-Warning "Build run details not read: $($_.Exception.Message)" }
  }
  $ids = @($releaseFill | ForEach-Object { [string]$_.Id } | Sort-Object -Unique)
  for ($i = 0; $i -lt $ids.Count; $i += 50) {
    $chunk = $ids[$i..([Math]::Min($i + 49, $ids.Count - 1))] -join ','
    try {
      (Invoke-RestMethod -Uri "$vsrm/$($p)/_apis/release/releases?releaseIdFilter=$chunk&`$top=100&api-version=7.1" -Headers $H).value |
        ForEach-Object { $info["r$($_.id)"] = @{ By = $_.createdBy.displayName; Reason = [string]$_.reason } }
    } catch { Write-Warning "Release details not read: $($_.Exception.Message)" }
  }
  foreach ($f in $buildFill)   { $x = $info["b$($f.Id)"]; if ($x) { $f.Row.LatestRunBy = $x.By; $f.Row.LatestRunReason = $x.Reason } }
  foreach ($f in $releaseFill) { $x = $info["r$($f.Id)"]; if ($x) { $f.Row.LatestRunBy = $x.By; $f.Row.LatestRunReason = $x.Reason } }

  # --- Node.js warnings in the latest run (optional, -NodeWarnings) ---
  if ($NodeWarnings) {
    $sinceText = if ($Since) { ([datetime]$Since).ToString('yyyy-MM-dd') } else { '' }
    $bT = @($buildFill   | Where-Object { -not $sinceText -or $_.Row.LatestRunDate -ge $sinceText })
    $rT = @($releaseFill | Where-Object { -not $sinceText -or $_.Row.LatestRunDate -ge $sinceText })
    Write-Host "  Checking $($bT.Count + $rT.Count) latest runs for Node.js warnings..."
    $b = $base; $v = $vsrm; $hd = $H; $pp = $p

    $rowMap = @{}
    $bIn = @(foreach ($f in $bT) { $k = "b$($f.Id)"; $rowMap[$k] = $f.Row; [pscustomobject]@{ Key = $k; Id = [string]$f.Id } })
    $rIn = @(foreach ($f in $rT) { $k = "r$($f.Id)"; $rowMap[$k] = $f.Row; [pscustomobject]@{ Key = $k; Id = [string]$f.Id } })

    $res = @($bIn | ForEach-Object -ThrottleLimit 8 -Parallel {
      $f = $_
      try {
        $tl = Invoke-RestMethod -Uri "$($using:b)/$($using:pp)/_apis/build/builds/$($f.Id)/timeline?api-version=7.1" -Headers $using:hd
        $msgs = @($tl.records | ForEach-Object { $_.issues } |
          Where-Object { $_ -and $_.type -eq 'warning' -and $_.message -match 'node' } | ForEach-Object { $_.message })
        $agents = @($tl.records | Where-Object { $_.type -eq 'Job' -and $_.workerName } | ForEach-Object { $_.workerName })
        $tasks  = @($tl.records | Where-Object { $_.type -eq 'Task' -and $_.task } |
          ForEach-Object { "$($_.task.id)|$($_.task.version)|$($_.task.name)" })
        [pscustomobject]@{ Key = $f.Key; Msgs = $msgs; Agents = $agents; Tasks = $tasks; Err = $null }
      } catch { [pscustomobject]@{ Key = $f.Key; Msgs = @(); Agents = @(); Tasks = @(); Err = $_.Exception.Message } }
    })

    $res += @($rIn | ForEach-Object -ThrottleLimit 8 -Parallel {
      $f = $_
      try {
        $rel = Invoke-RestMethod -Uri "$($using:v)/$($using:pp)/_apis/release/releases/$($f.Id)?api-version=7.1" -Headers $using:hd
        $jobs = @(foreach ($e in @($rel.environments)) { foreach ($ds in @($e.deploySteps)) { foreach ($ph in @($ds.releaseDeployPhases)) {
          foreach ($dj in @($ph.deploymentJobs)) { $dj } } } })
        $msgs = @(foreach ($dj in $jobs) {
            foreach ($i in (@($dj.job.issues) + @($dj.tasks | ForEach-Object { $_.issues }))) {
              if ($i -and "$($i.issueType)$($i.type)" -match 'warning' -and $i.message -match 'node') { $i.message }
            } })
        $agents = @($jobs | ForEach-Object { $_.job.agentName } | Where-Object { $_ })
        $tasks  = @($jobs | ForEach-Object { $_.tasks } | Where-Object { $_ -and $_.task } |
          ForEach-Object { "$($_.task.id)|$($_.task.version)|$($_.task.name)" })
        [pscustomobject]@{ Key = $f.Key; Msgs = $msgs; Agents = $agents; Tasks = $tasks; Err = $null }
      } catch { [pscustomobject]@{ Key = $f.Key; Msgs = @(); Agents = @(); Tasks = @(); Err = $_.Exception.Message } }
    })

    $res  = @($res | Where-Object { $_ })
    $errs = @($res | Where-Object { $_.Err })
    Write-Host "  Got details for $($res.Count - $errs.Count) runs; $($errs.Count) could not be read."
    if ($errs.Count) { Write-Warning "First error: $($errs[0].Err)" }

    foreach ($x in $res) {
      $row = $rowMap[$x.Key]
      if (-not $row) { continue }
      if ($x.Err) {
        $why = switch -Regex ($x.Err) {
          '404|NotFound|does not exist|not found' { 'run deleted (retention)'; break }
          '401|403|Unauthorized|Forbidden|permission' { 'no permission'; break }
          '429|TooManyRequests|throttl'            { 'throttled, rerun'; break }
          'timed out|timeout'                       { 'timed out, rerun'; break }
          default { ($x.Err -replace '\s+', ' ').Substring(0, [Math]::Min(80, ($x.Err -replace '\s+', ' ').Length)) }
        }
        $row.NodeWarnings = "(could not read run: $why)"
        continue
      }
      $row.NodeWarnings     = [string]@($x.Msgs).Count
      $row.NodeWarningTasks = Get-NodeSummary $x.Msgs
      $row.LatestRunAgents  = (@($x.Agents) | Sort-Object -Unique) -join '; '

      # Node version named in this run's warnings, per task name (fallback when the definition has no handler info)
      $warnNode = @{}
      $taskNames = @(@($x.Tasks) | ForEach-Object { ($_ -split '\|', 3)[2] } | Where-Object { $_ } | Sort-Object -Unique)
      foreach ($m in @($x.Msgs)) {
        $m = [string]$m
        if ($m -notmatch 'Node(?:\.js)?\s*(?:version\s*)?\(?\s*(?<n>\d+)') { continue }
        $n = [int]$Matches.n
        foreach ($tn in $taskNames) {
          if ($m -match "(?i)\b$([regex]::Escape($tn))\b") {
            if (-not $warnNode.ContainsKey($tn) -or $n -lt $warnNode[$tn]) { $warnNode[$tn] = $n }
          }
        }
      }

      $runners = @(foreach ($tk in (@($x.Tasks) | Sort-Object -Unique)) {
          $parts = $tk -split '\|', 3
          $best  = Get-BestRunner (Get-TaskHandlers $parts[0] $parts[1])
          $node = $best.Node; $label = $best.Label
          if (-not $node -and $warnNode.ContainsKey($parts[2])) { $node = $warnNode[$parts[2]]; $label = "Node $node (from warning)" }
          [pscustomobject]@{ Task = "$($parts[2])@$($parts[1])"; Node = $node; Label = $label }
        })
      if ($runners.Count) {
        $row.TaskRunners = ($runners | Sort-Object @{ e = { if ($_.Node) { $_.Node } else { 99 } } }, Task |
          ForEach-Object { "$($_.Task)=$($_.Label)" }) -join '; '
        $min = @($runners | Where-Object { $_.Node }) | Measure-Object Node -Minimum
        $row.LowestNodeRunner = if ($min.Count) { "Node $($min.Minimum)" }
          elseif (@($runners | Where-Object { $_.Label -ne 'unknown' }).Count) { '(no Node tasks)' }
          else { '(runner unknown)' }
      }
    }
    $withH = @($taskCat.Values | Where-Object { @($_).Count -gt 0 }).Count
    Write-Host "  Task definitions loaded: $($taskLoaded.Count) tasks, $withH of $($taskCat.Count) versions have runner info ($script:taskMissing not readable)."
    $hit = @($res | Where-Object { @($_.Msgs).Count -gt 0 }).Count
    Write-Host "  $hit latest runs have Node.js warnings."
    $rowMap.Values | Where-Object { $_.LowestNodeRunner } | Group-Object LowestNodeRunner |
      Sort-Object Name | ForEach-Object { Write-Host ("    Lowest runner {0}: {1} runs" -f $_.Name, $_.Count) }
  }
}

$sorted = @($rows | Sort-Object Project, Folder, Pipeline)

# --- Unique task runners: every distinct entry from the TaskRunners column, one per row ---
$uniqueRunners = @($sorted | Where-Object { $_.TaskRunners } |
  ForEach-Object { $_.TaskRunners -split ';\s*' } | Where-Object { $_ } |
  Sort-Object -Unique | ForEach-Object { [pscustomobject]@{ TaskRunner = $_ } })

# --- Pool summary ---
$summary = @($sorted | ForEach-Object {
    $r = $_
    "$($r.PoolsUsed); $($r.ConfiguredPools)" -split ';\s*' | Where-Object { $_ } | Sort-Object -Unique |
      ForEach-Object { [pscustomobject]@{ Pool = $_; Name = "$($r.Project) | $($r.Pipeline)" } }
  } | Group-Object Pool | Sort-Object Count -Descending | ForEach-Object {
    [pscustomobject]@{
      Pool          = $_.Name
      PipelineCount = $_.Count
      Pipelines     = ($_.Group.Name | Sort-Object) -join "`n"
    }
  })

# --- Excel output ---
function Write-ComSheet($ws, [string]$name, $data) {
  $ws.Name = $name
  $data = @($data)
  if ($data.Count -eq 0) { $ws.Cells.Item(1, 1).Value2 = '(no rows)'; return }
  $cols = @($data[0].PSObject.Properties.Name)
  $arr  = New-Object 'object[,]' ($data.Count + 1), $cols.Count
  for ($c = 0; $c -lt $cols.Count; $c++) { $arr[0, $c] = $cols[$c] }
  for ($r = 0; $r -lt $data.Count; $r++) {
    for ($c = 0; $c -lt $cols.Count; $c++) { $arr[($r + 1), $c] = [string]$data[$r].($cols[$c]) }
  }
  $ws.Cells.NumberFormat = '@'   # keep everything as text (folder paths, names)
  $rng = $ws.Range($ws.Cells.Item(1, 1), $ws.Cells.Item($data.Count + 1, $cols.Count))
  $rng.Value2 = $arr
  $rng.Font.Name = 'Arial'
  $rng.Font.Size = 10
  $rng.VerticalAlignment = -4160          # top
  $hdr = $ws.Range($ws.Cells.Item(1, 1), $ws.Cells.Item(1, $cols.Count))
  $hdr.Font.Bold = $true
  $hdr.Interior.Color = 0x47331F          # dark navy (BGR)
  $hdr.Font.Color = 0xFFFFFF
  # Make the PipelineUrl column clickable
  $urlCol = [array]::IndexOf($cols, 'PipelineUrl') + 1
  if ($urlCol -gt 0) {
    for ($r = 0; $r -lt $data.Count; $r++) {
      $u = [string]$data[$r].PipelineUrl
      if ($u) { [void]$ws.Hyperlinks.Add($ws.Cells.Item($r + 2, $urlCol), $u, '', $u, $u) }
    }
  }
  [void]$rng.AutoFilter()
  [void]$ws.Columns.AutoFit()
  for ($c = 1; $c -le $cols.Count; $c++) {
    if ($ws.Columns.Item($c).ColumnWidth -gt 70) { $ws.Columns.Item($c).ColumnWidth = 70; $ws.Columns.Item($c).WrapText = $true }
  }
  try {
    $ws.Activate()
    $ws.Application.ActiveWindow.SplitRow = 1
    $ws.Application.ActiveWindow.FreezePanes = $true
  } catch { }
}

function Export-Workbook($Path) {
  $full = $ExecutionContext.SessionState.Path.GetUnresolvedProviderPathFromPSPath($Path)
  if (Test-Path $full) { Remove-Item $full -Force }

  if (Get-Module -ListAvailable -Name ImportExcel) {
    Import-Module ImportExcel
    $sorted  | Export-Excel $full -WorksheetName 'Pipelines'    -AutoSize -AutoFilter -FreezeTopRow -BoldTopRow
    $summary | Export-Excel $full -WorksheetName 'Pool Summary' -AutoSize -AutoFilter -FreezeTopRow -BoldTopRow
    if ($uniqueRunners.Count) { $uniqueRunners | Export-Excel $full -WorksheetName 'UniqueTaskRunners' -AutoSize -AutoFilter -FreezeTopRow -BoldTopRow }
    return $full
  }

  $xl = $null
  try { $xl = New-Object -ComObject Excel.Application } catch { return $null }
  try {
    $xl.Visible = $false
    $xl.DisplayAlerts = $false
    $wb = $xl.Workbooks.Add()
    while ($wb.Worksheets.Count -lt 3) { [void]$wb.Worksheets.Add([Type]::Missing, $wb.Worksheets.Item($wb.Worksheets.Count)) }
    while ($wb.Worksheets.Count -gt 3) { $wb.Worksheets.Item($wb.Worksheets.Count).Delete() }
    Write-ComSheet $wb.Worksheets.Item(1) 'Pipelines'         $sorted
    Write-ComSheet $wb.Worksheets.Item(2) 'Pool Summary'      $summary
    Write-ComSheet $wb.Worksheets.Item(3) 'UniqueTaskRunners' $uniqueRunners
    $wb.Worksheets.Item(1).Activate()
    $wb.SaveAs($full, 51)   # 51 = .xlsx
    $wb.Close($false)
    return $full
  } finally {
    $xl.Quit()
    [void][Runtime.InteropServices.Marshal]::ReleaseComObject($xl)
  }
}

$saved = Export-Workbook $OutFile
if (-not $saved) {
  $saved = [IO.Path]::ChangeExtension($ExecutionContext.SessionState.Path.GetUnresolvedProviderPathFromPSPath($OutFile), '.csv')
  $sorted | Export-Csv $saved -NoTypeInformation
  Write-Warning "Excel and the ImportExcel module weren't available, so the result was saved as CSV."
}

# --- Console summary ---
$summary | Format-Table Pool, PipelineCount -AutoSize
foreach ($projId in $projInfo.Keys) {
  $n = @($sorted | Where-Object Project -eq $projInfo[$projId]).Count
  Write-Host ("{0}: {1} pipelines; job history goes back to {2}" -f $projInfo[$projId], $n, $oldest[$projId])
}
Write-Host "Saved: $saved"   
```

