{% raw %}
```yaml

```
{% endraw %}

```ps1
<#
.SYNOPSIS
  Lists every pipeline in one or more Azure DevOps projects and the agent pool(s) each uses,
  and saves the result as an Excel workbook.

.DESCRIPTION
  Workbook sheets:
    Pipelines     - one row per pipeline definition (all folders, including never-run/disabled ones)
    Pool Summary  - one row per pool: how many pipelines use it, and which

  Pipelines columns:
    Project, Type, Pipeline, Folder
    PoolsUsed       - pools the pipeline's jobs actually ran on (from each pool's job history)
    ConfiguredPools - pools set on the definition (classic build phases, classic release stages)
    DefaultPool     - YAML only: the default queue on the pipeline settings (the YAML 'pool:' overrides it)
    Source          - YAML only: repo and YAML file path

  Excel output uses the ImportExcel module if installed, otherwise Excel itself (must be installed).
  If neither is available it falls back to a .csv with the same columns.

.EXAMPLE
  .\Get-PipelinePools.ps1 -Project "RxI-Integration"
  .\Get-PipelinePools.ps1 -Project "RxI-DevOps-Deployment","RxI-Integration","RxR-SCM"
  .\Get-PipelinePools.ps1 -Project "RxR-SCM" -OutFile .\scm-pools.xlsx -JobHistory 10000
#>
param(
  [Parameter(Mandatory)][string[]]$Project,
  [string]$Org = 'Rx-Renewal',
  [int]$JobHistory = 5000,
  [string]$OutFile
)
$ErrorActionPreference = 'Stop'
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

function New-Set { [System.Collections.Generic.HashSet[string]]::new([StringComparer]::OrdinalIgnoreCase) }
function Join-Set($s) { if ($s -and $s.Count) { ($s | Sort-Object) -join '; ' } else { '' } }

# --- Resolve projects ---
$projInfo = @{}   # projectId -> name
foreach ($name in $Project) {
  $pr = Invoke-RestMethod -Uri "$base/_apis/projects/$([uri]::EscapeDataString($name))?api-version=7.1" -Headers $H
  $projInfo[$pr.id] = $pr.name
}

# --- Observed: which pools each pipeline's jobs ran on (read once for all projects) ---
Write-Host "Reading job history from every pool in $Org (this can take a minute)..."
$observed = @{}   # "projectId|planType|definitionId" -> set of pool names
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
    if (-not $observed.ContainsKey($key)) { $observed[$key] = New-Set }
    [void]$observed[$key].Add($pool.name)
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
  $defs = Get-AdoAll "$base/$($p)/_apis/build/definitions?includeAllProperties=true&queryOrder=definitionNameAscending&api-version=7.1"
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
    $rows.Add([pscustomobject]@{
      Project         = $projName
      Type            = if ($isYaml) { 'YAML' } else { 'Classic build' }
      Pipeline        = $d.name
      Folder          = $d.path
      PoolsUsed       = Join-Set $observed["$projId|Build|$($d.id)"]
      ConfiguredPools = Join-Set $cfg
      DefaultPool     = if ($isYaml) { $defaultPool } else { '' }
      Source          = if ($isYaml) { "$($d.repository.name):$($d.process.yamlFilename)" } else { '' }
    })
  }

  # Classic release pipelines
  try {
    $rdefs = Get-AdoAll "$vsrm/$($p)/_apis/release/definitions?api-version=7.1"
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
    $rows.Add([pscustomobject]@{
      Project         = $projName
      Type            = 'Classic release'
      Pipeline        = $full.name
      Folder          = $full.path
      PoolsUsed       = Join-Set $observed["$projId|Release|$($rd.id)"]
      ConfiguredPools = Join-Set $cfg
      DefaultPool     = ''
      Source          = ''
    })
  }
}

$sorted = @($rows | Sort-Object Project, Folder, Pipeline)

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
  $full = [IO.Path]::GetFullPath($Path)
  if (Test-Path $full) { Remove-Item $full -Force }

  if (Get-Module -ListAvailable -Name ImportExcel) {
    Import-Module ImportExcel
    $sorted  | Export-Excel $full -WorksheetName 'Pipelines'    -AutoSize -AutoFilter -FreezeTopRow -BoldTopRow
    $summary | Export-Excel $full -WorksheetName 'Pool Summary' -AutoSize -AutoFilter -FreezeTopRow -BoldTopRow
    return $full
  }

  $xl = $null
  try { $xl = New-Object -ComObject Excel.Application } catch { return $null }
  try {
    $xl.Visible = $false
    $xl.DisplayAlerts = $false
    $wb = $xl.Workbooks.Add()
    while ($wb.Worksheets.Count -lt 2) { [void]$wb.Worksheets.Add([Type]::Missing, $wb.Worksheets.Item($wb.Worksheets.Count)) }
    while ($wb.Worksheets.Count -gt 2) { $wb.Worksheets.Item($wb.Worksheets.Count).Delete() }
    Write-ComSheet $wb.Worksheets.Item(1) 'Pipelines'    $sorted
    Write-ComSheet $wb.Worksheets.Item(2) 'Pool Summary' $summary
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
  $saved = [IO.Path]::ChangeExtension([IO.Path]::GetFullPath($OutFile), '.csv')
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
