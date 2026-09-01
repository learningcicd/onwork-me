{% raw %}
```yaml


```
{% endraw %}


# Timeouts in this pipeline — a field guide

This pipeline has **two fundamentally different kinds of timeout**, and mixing them up is the most common source of confusion when a run "seems stuck." This doc explains both, lists every timeout value in the pipeline, and explains the one Azure DevOps rule that trips people up most: **job-level timeout vs. task-level timeout, and which one actually governs.**

## The two kinds of timeout

### 1. Work timeouts — "how long can this job run before ADO assumes it's hung?"

These sit on jobs that do real work on a real agent: downloading an artifact, running a PowerShell script, calling the Azure Functions deploy API. If a job runs past this limit, ADO kills it and fails the job — the assumption is that something has hung (a stuck download, an API call that never returns), not that a human needs to make a decision.

### 2. Approval timeouts — "how long do we wait for a human to click Approve/Reject?"

These sit on the `ManualValidation@0` task (and the equivalent inside the shared `job-manual-approval.yml@rxiPipelineTemplate`). Nothing is "running" while this waits — the job just sits idle on the agentless `server` pool, costing nothing, until someone responds or the clock runs out.

**These two numbers are set completely independently and answer completely different questions.** A 190-minute work timeout on a deploy job has nothing to do with how long an approval gate waits.

## The critical rule: job timeout vs. task timeout

Every `job:` in this pipeline can carry its own `timeoutInMinutes`. Some jobs *also* contain a `task:` (like `ManualValidation@0`) that has its **own**, separate `timeoutInMinutes`. When both exist:

> **The JOB's timeout is a hard ceiling that applies no matter what the task's own timeout says.** If the job timeout is shorter than the task timeout, the job gets killed first — the task's timeout never gets a chance to matter.

This is exactly why `diffApprovalJob`'s job-level timeout is set to `1440` (1 day) even though the task-level timeout inside it is commented out (unset). If we'd left the job-level wrapper at a low number like 60, the gate would have been silently cut off at an hour, regardless of the fact that the task itself was configured to wait indefinitely.

**Microsoft's own guidance:** the job's `timeoutInMinutes` must be *greater than* the task's `timeoutInMinutes` for the task's timeout to be the one that actually governs.

## Another rule that matters here: Azure DevOps' default is 60 minutes, not "unlimited"

If you don't specify `timeoutInMinutes` on a job at all, Azure DevOps does **not** let it run forever — it silently applies a **default of 60 minutes**. This is true for jobs on real agents and for agentless (`server` pool) jobs alike. The maximum you can raise an agentless job to is **30 days (43,200 minutes)**.

So "not setting a timeout" is never a way to get infinite waiting — it's a way to get exactly 60 minutes, which is almost always shorter than you'd want for a real approval gate. Every timeout in this pipeline is set explicitly for this reason.

## Every timeout in this pipeline

### Non-prod environments (`dev`, `e2e-01`, `e2e-02`, `perf-01`, `prodfix-01`) — one set of three jobs, repeated per environment

| Job | Type | Timeout | What it means |
|---|---|---|---|
| `prepJob` | Work (real agent) | **190 min** (~3h 10m) | Download artifact, capture live settings, build new settings, print diff. Matches helm's equivalent `deployJob` timeout value. |
| `diffApprovalJob` | Approval (agentless) | **Job: 1440 min (1 day)** · Task: *unset* | Waits for a human to approve/reject after reviewing the diff. Task-level timeout is deliberately commented out to match how helm's reference pipeline handles this gate (no explicit timeout — relies on default behavior). The 1-day job-level ceiling exists only so this doesn't silently fall back to the 60-minute ADO default. |
| `deployJob` | Work (real agent) | **190 min** | The actual code/settings deploy. Matches helm exactly. |

**What you'll see if a run sits for hours:** if `Prepare & Diff` shows ✅ done in under a minute, but `Approve deploy after diff review` shows a blue clock and has been sitting for, say, 13 hours — **this is expected**, not stuck. It's waiting on a human, and won't time out until the 24-hour mark.

### Break-Glass Approval

| Job | Type | Timeout | What it means |
|---|---|---|---|
| `approvalJob` (shared template) | Approval (agentless) | *Unset — commented out* | Matches helm, which never passes a timeout to this shared template. Falls back to whatever `job-manual-approval.yml@rxiPipelineTemplate`'s internal default is — observed in practice to be noticeably shorter than 24 hours, since Break-Glass gates typically show as failed/rejected well before the non-prod gates time out. |

### QE Review & Approval for Prod

| Job | Type | Timeout | What it means |
|---|---|---|---|
| `prepProdJob` | Work (real agent) | **2880 min (2 days)** | Prod's Prepare & Diff, run fresh inside the QE stage. |
| `qe_final_approval` (shared template) | Approval (agentless) | **2880 min (2 days)** | Matches helm's QE gates exactly. |

### Deploy to Prod

| Job | Type | Timeout | What it means |
|---|---|---|---|
| `prepProdDiffJob` | Work (real agent) | **2880 min (2 days)** | A second, independent fresh Prod diff, run again right before the prod approval — real time has passed since the QE gate. |
| `prod_approval` (shared template) | Approval (agentless) | **1440 min (1 day)** | Matches helm's `DeployToProd` gate exactly. |
| `prod_deploy` | Work (real agent) | **190 min** | The actual prod deploy. |

### Deploy to prodfix-01 (post-prod)

| Job | Type | Timeout | What it means |
|---|---|---|---|
| `prepProdfixDiffJob` | Work (real agent) | **2880 min (2 days)** | prodfix-01's Prepare & Diff. |
| `prodfix_approval` (shared template) | Approval (agentless) | **1440 min (1 day)** | Matches helm's `post_deploy_stage` prodfix gate exactly. |
| `prodfix_deploy` | Work (real agent) | **190 min** | The actual prodfix-01 deploy. |

## Quick answers to "is this stuck?"

- **A work job (Prepare & Diff, Deploy) has been running for longer than its listed minutes above** → genuinely worth investigating. Something is likely hung (agent lost connectivity, a stuck download, an Azure API call that never returned).
- **An approval job (Approve deploy after diff review, QE Review & Approval, Approve Prod Deploy, prodfix approval) shows a blue clock and hasn't hit its listed ceiling yet** → not stuck. It's waiting for a human. Check who's in the relevant approvers group and nudge them, or approve/reject it yourself if you're authorized.
- **A stage shows "Not started" even though one of its dependency stages already succeeded** → this is expected Azure DevOps behavior, not a bug. A stage's `dependsOn` always waits for **every** listed dependency to reach a terminal state (succeeded, failed, *or* rejected/timed-out) before it even evaluates whether to run — there's no way in native ADO YAML to say "start as soon as the first of these finishes." `QEFinalApproval`, for example, lists all five non-prod stages in `dependsOn`; even though its `condition` only requires **one** of them to have succeeded, the stage itself won't begin until all five have finished one way or another.

## Where each helm-matched value came from

If you ever need to re-verify these against the reference pipeline (`refernce-helm-cd.yaml`):

| Our value | Helm source |
|---|---|
| `deployJob` / `prod_deploy` / `prodfix_deploy` = 190 min | Helm's nonprod `deployJob`, line ~120 |
| `qe_final_approval` = 2880 min | Helm's `QEFinalApproval` stage gates, lines ~191, ~225 |
| `prod_approval` = 1440 min | Helm's `DeployToProd` approval, line ~265 |
| `prodfix_approval` = 1440 min | Helm's `post_deploy_stage` prodfix approval, line ~309 |
| Nonprod `diffApprovalJob` task timeout = unset | Helm's per-env `approvalJob` also has no explicit timeout |
| Break-Glass timeout = unset | Helm's Break-Glass gate also has no explicit timeout |

Values **not** found anywhere in helm (chosen independently, not matched to a reference):
- `prepJob`, `prepProdJob`, `prepProdDiffJob`, `prepProdfixDiffJob` — helm doesn't have a directly equivalent standalone "prepare and diff" job with its own timeout; ours are set to 190 min (nonprod) or 2880 min (prod-side) as reasonable working ceilings.
- `diffApprovalJob`'s **job-level** 1440-minute wrapper — this exists purely to stop ADO's 60-minute default from silently capping the gate now that its task-level timeout is unset. Helm doesn't need an equivalent because it doesn't use this inline-task pattern for its per-env gate.
