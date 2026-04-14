---
title: "Entity Properties in Jira Automation: Idempotency, Locks, and Wait-for-API Patterns"
date: 2026-01-14T00:00:00-03:00
draft: false
translationKey: "entity-properties-automation"
tags: ["jira-automation", "entity-properties", "idempotency", "locks", "rest-api", "patterns"]
---

Manual triggers and web requests are a perfect recipe for duplicate executions.

Someone clicks the rule twice. Two admins run it "just to be sure". Or two automations race each other and you end up with duplicated comments, repeated API calls, or inconsistent data.

This article shows production-friendly patterns that use **Entity Properties** as a lightweight state store inside Jira Automation.

The goal is simple: make your rules safer when they need to run only once, coordinate with external APIs, or avoid getting stuck in bad states.

The patterns covered here are:

- per-issue locks
- idempotency keys
- wait-for-API orchestration
- stale lock takeover
- safe retries with an attempts counter

---

## What are Entity Properties

Entity Properties are small pieces of structured data that Jira can store on entities such as issues.

In practice, they work well as a lightweight state store for automation rules.

That makes them useful when you need to remember things like:

- whether a rule is already running
- whether an external action already happened
- which job ID was returned by an API
- how many retry attempts already happened
- what the last error was

---

## The core idea: store state on the issue

Many automation problems happen because the rule has no memory of a previous execution.

A practical way to reduce that problem is to store explicit state directly on the issue.

Example JSON stored as an issue property:

```json
{
  "status": "running",
  "runId": "rule-123-exec-456",
  "startedAt": "2026-01-14T03:00:00Z",
  "attempts": 1,
  "lastError": null
}
```

Every run should be able to answer a few important questions:

- Is something already running? (lock)
- Did we already complete this operation? (idempotency)
- If it failed, should we retry? (retry policy)
- If it is still marked as running, is that state stale? (timeout or takeover)

---

## Pattern 1: Manual trigger anti-duplication (per-issue lock)

**Goal:** if the rule is triggered twice, the second execution exits immediately.

### Property key

Use a predictable key, for example:

- `automation.lock.manualSync`

### Rule flow

1. **Trigger:** Manual trigger
2. **Condition (Advanced compare):** lock is empty
3. **Action:** Set entity property to create the lock
4. Do the actual work
5. Update the property to mark the run as done

### Condition example

- First value: `{{issue.properties.automation.lock.manualSync}}`
- Condition: **is empty**

### Create the lock

```json
{
  "status": "running",
  "runId": "{{rule.id}}-{{executionId}}",
  "startedAt": "{{now}}"
}
```

### Mark the work as completed

```json
{
  "status": "done",
  "runId": "{{rule.id}}-{{executionId}}",
  "finishedAt": "{{now}}"
}
```

### Why it works

The first execution writes a state flag. Parallel executions read that flag and stop.

---

## Pattern 2: Idempotency key (do not repeat side effects)

Locks prevent concurrency. Idempotency prevents repeating the same operation later, even if the lock is no longer present.

This is useful for actions that should happen only once per issue, such as:

- creating something in an external system
- provisioning an account or resource
- calling a create endpoint in an external API

### Property key

- `automation.idempotency.createExternal`

### Rule flow

1. If `{{issue.properties.automation.idempotency.createExternal}}` is **not empty**, stop the rule
2. Otherwise, perform the operation
3. Store the completion state

### Example completion state

```json
{
  "done": true,
  "doneAt": "{{now}}",
  "by": "{{initiator.accountId}}"
}
```

### Useful tip

If the external API also supports idempotency headers, send a stable key such as:

- `{{issue.key}}:createExternal`

That way, even if Jira retries the rule, the external system can still reject duplicate creation requests safely.

---

## Pattern 3: Wait-for-API without duplicating calls

Many APIs are not truly synchronous.

A common pattern looks like this:

- `POST /jobs` returns immediately with a `jobId`
- later you poll `GET /jobs/{jobId}` until the job becomes `DONE`

A safer approach is to split the logic into **two rules**:

- a starter rule
- a poller rule

### Step A: Starter rule

- **Trigger:** manual trigger or issue event
- **Guards:** lock + idempotency
- **Action:** send a web request to create the external job

Then store the returned job ID.

### Property key

- `automation.job.externalSync`

### Example stored state

```json
{
  "status": "started",
  "jobId": "{{webResponse.body.jobId}}",
  "startedAt": "{{now}}",
  "attempts": 1
}
```

You can also add a comment or update a field such as `Sync status = Running`.

### Step B: Poller rule

Possible triggers:

- a scheduled rule every few minutes with a JQL filter
- or a rule triggered by a field update, if you want tighter control

### Example JQL

```text
issue.property[automation.job.externalSync].status = "started"
```

Then:

- send `GET /jobs/{{issue.properties.automation.job.externalSync.jobId}}`
- if the API returns `DONE`, update the issue and mark the property as done
- if it returns `FAILED`, store the error and decide whether to retry
- if it is still running, do nothing and let the schedule run again later

### Mark completed

```json
{
  "status": "done",
  "jobId": "{{issue.properties.automation.job.externalSync.jobId}}",
  "finishedAt": "{{now}}",
  "result": "{{webResponse.body.result}}"
}
```

### Mark failed

```json
{
  "status": "failed",
  "jobId": "{{issue.properties.automation.job.externalSync.jobId}}",
  "failedAt": "{{now}}",
  "lastError": "{{webResponse.status}} - {{webResponse.body}}"
}
```

---

## Pattern 4: Stale lock takeover

Runs can fail, time out, or stop unexpectedly.

When that happens, a property may stay in `running` forever and block future executions.

To avoid that, define a timeout policy.

Example:

- if the lock is older than 10 minutes, consider it stale and allow takeover

### Practical decision flow

- if the lock is empty, proceed
- if the lock is running but stale, overwrite it and proceed
- otherwise, stop the rule

### Practical tip

Start with a conservative timeout, such as 10 to 30 minutes.

---

## Pattern 5: Safe retries with an attempts counter

Retries can be useful, but they are dangerous when the rule causes side effects.

Instead of retrying blindly, make retries explicit and stateful.

### Property key

- `automation.retry.externalSync`

### Example retry state

```json
{
  "status": "failed",
  "attempts": 2,
  "lastAttemptAt": "{{now}}",
  "lastStatus": 502
}
```

### Example retry policy

- if attempts >= 3, stop and notify someone
- if the last status was 429, 502, 503, or 504, retry
- otherwise, stop and require manual review

---

## A naming convention that scales

If you plan to use this pattern in more than one rule, keep property keys predictable.

A practical convention is:

- `automation.lock.<ruleKey>`
- `automation.idempotency.<operation>`
- `automation.job.<integrationName>`
- `automation.retry.<operation>`

This makes the properties easier to understand, search, and maintain.

---

## Quick checklist before you ship

Before moving this kind of automation into production, check the basics:

- Does the rule exit safely if triggered twice?
- Does it avoid repeating create operations?
- Are failures stored somewhere useful?
- Is there a stale timeout so the rule does not stay blocked forever?
- Do retries have a clear cap?

If you implement just **Pattern 1 (lock)** and **Pattern 2 (idempotency)**, you already remove a large part of the duplicate execution risk.

---

## Final thoughts

Entity Properties are simple, but very powerful when you need automation rules to behave more like real systems.

They help Jira Automation keep memory of what already happened, which is exactly what many advanced rules are missing.

If your rules trigger APIs, create external resources, or depend on long-running jobs, this pattern is worth learning.
