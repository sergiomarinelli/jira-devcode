---
title: "Entity Properties in Jira Automation: Idempotency, Locks, and Wait-for-API Patterns"
date: 2026-01-14T00:00:00-03:00
draft: false
tags: ["jira-automation", "entity-properties", "idempotency", "locks", "rest-api", "patterns"]
---

Manual triggers plus web requests are a perfect recipe for duplicate executions.

People click the rule twice. Two admins run it "just to be sure". Or two automations race and you end up with duplicated comments, repeated API calls, or inconsistent data.

This article shows production-friendly patterns using Entity Properties as a lightweight state store to implement:

- **Idempotency**: the same operation will not run twice for the same issue
- **Per-issue locks**: only one execution at a time per issue
- **Async orchestration**: start an API job now, finish it later safely
- **Safe retries**: retry only when needed, without duplicating side effects

Entity properties are key/value JSON metadata stored on Jira entities. Automation can write and read them and use them in conditions.

---

## What are Entity Properties
Entity properties are JSON blobs stored against Jira entities like issues, projects, and users. In Automation rules, you can:

- Set them via **Set entity property**
- Read them using smart values like `{{issue.properties.myKey}}`

Think of them as a small state record attached to the issue.

---

## The core idea: store state on the issue
Most automation bugs come from not knowing the state of a previous run. Fix that by storing explicit state.

Example JSON stored per issue:

```json
{
  "status": "running",
  "runId": "rule-123-exec-456",
  "startedAt": "2026-01-14T03:00:00Z",
  "attempts": 1,
  "lastError": null
}
```

Every run should answer:

- Is something already running? (lock)
- Did we already complete? (idempotency)
- If failed, should we retry? (retry policy)
- If running, is it stale? (timeout / takeover)

---

## Pattern 1: Manual trigger anti-duplication (per-issue lock)
**Goal:** if the rule is clicked twice, the second execution exits immediately.

### Property key
Use something predictable, for example:

- `automation.lock.manualSync`

### Rule flow
1) **Trigger:** Manual trigger

2) **Condition (Advanced compare):** lock is empty  
- First value: `{{issue.properties.automation.lock.manualSync}}`  
- Condition: **is empty**

3) **Action:** Set entity property (create lock)  
- Entity: Issue  
- Property key: `automation.lock.manualSync`  
- Value:

```json
{
  "status": "running",
  "runId": "{{rule.id}}-{{executionId}}",
  "startedAt": "{{now}}"
}
```

4) **Do your work** (edits, transitions, web requests)

5) **Release lock** (mark done)  
Set the same property to:

```json
{
  "status": "done",
  "runId": "{{rule.id}}-{{executionId}}",
  "finishedAt": "{{now}}"
}
```

**Why it works:** the first execution sets a flag; parallel executions see the flag and stop.

---

## Pattern 2: Idempotency key (do not repeat side effects)
Locks prevent concurrency. Idempotency prevents repeating the same operation later, even if the lock is cleared.

Use this for operations that must happen exactly once per issue, like:

- creating something in an external system
- provisioning a resource/account
- calling a "create" endpoint

### Property key
- `automation.idempotency.createExternal`

### Flow
1) **If** `{{issue.properties.automation.idempotency.createExternal}}` is **not empty** → **Stop rule**
2) Else → perform the operation
3) Set the property to remember completion:

```json
{
  "done": true,
  "doneAt": "{{now}}",
  "by": "{{initiator.accountId}}"
}
```

### Tip: align with external idempotency (if available)
If your API supports idempotency headers, send a stable key like:

- `{{issue.key}}:createExternal`

That way, even if Jira retries, the external system will not create duplicates.

---

## Pattern 3: Wait-for-API without duplicating calls (async orchestration)
Many APIs are not truly synchronous:

- `POST /jobs` returns immediately with a `jobId`
- later you poll `GET /jobs/{jobId}` until status becomes `DONE`

A safe approach is two-step orchestration: **Starter rule** + **Poller rule**.

### Step A: Starter rule (create job once)
- **Trigger:** manual trigger or issue event
- **Guards:** lock + idempotency (Patterns 1 and 2)
- **Action:** Send web request to create the job

Store the job id:

Property key: `automation.job.externalSync`

```json
{
  "status": "started",
  "jobId": "{{webResponse.body.jobId}}",
  "startedAt": "{{now}}",
  "attempts": 1
}
```

Optionally: add a comment or set a field like "Sync status = Running".

### Step B: Poller rule (continue later)
Trigger options:
- **Scheduled rule** every X minutes + JQL filter
- Or a rule triggered by a field change if you prefer controlling it via a custom field

**JQL idea:** only issues with a running job

```text
issue.property[automation.job.externalSync].status = "started"
```

Then:
- Send web request: `GET /jobs/{{issue.properties.automation.job.externalSync.jobId}}`
- If status == DONE → update issue and mark property as done
- If status == FAILED → store error and decide whether to retry
- If still running → do nothing and let the schedule run again

Mark completed:

```json
{
  "status": "done",
  "jobId": "{{issue.properties.automation.job.externalSync.jobId}}",
  "finishedAt": "{{now}}",
  "result": "{{webResponse.body.result}}"
}
```

Mark failed:

```json
{
  "status": "failed",
  "jobId": "{{issue.properties.automation.job.externalSync.jobId}}",
  "failedAt": "{{now}}",
  "lastError": "{{webResponse.status}} - {{webResponse.body}}"
}
```

---

## Pattern 4: Stale lock takeover (avoid being stuck forever)
Runs can crash or time out, leaving a lock in `running` forever.

Add a timeout policy:

- If lock is older than (for example) 10 minutes, consider it stale and allow takeover.

Store a timestamp in the lock (Pattern 1 already does). Then add logic at the start:

- If lock is empty → proceed
- Else if lock is running and stale → overwrite lock and proceed
- Else → stop

**Pragmatic tip:** keep your stale timeout conservative. Most teams start with 10 to 30 minutes.

---

## Pattern 5: Safe retries with an attempts counter
Retries are dangerous when the operation has side effects. Make retries explicit:

Property key: `automation.retry.externalSync`

```json
{
  "status": "failed",
  "attempts": 2,
  "lastAttemptAt": "{{now}}",
  "lastStatus": 502
}
```

Retry policy example:
- If attempts >= 3 → stop and notify
- If lastStatus is 429/502/503/504 → retry
- Otherwise → stop and require manual intervention

---

## Naming convention that scales
Keep keys predictable and searchable:

- `automation.lock.<ruleKey>`
- `automation.idempotency.<operation>`
- `automation.job.<integrationName>`
- `automation.retry.<operation>`

---

## Quick checklist before you ship
- Does the rule exit safely if triggered twice?
- Does it avoid repeating create operations?
- Are failures stored somewhere (property, comment, field)?
- Is there a stale timeout to avoid stuck locks?
- Do retries have a cap?

If you implement just Pattern 1 (lock) and Pattern 2 (idempotency), you already eliminate most duplicate execution pain.
