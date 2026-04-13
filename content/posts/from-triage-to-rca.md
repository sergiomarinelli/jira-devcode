+++
date = '2026-04-13T10:14:47-03:00'
draft = false
title = 'From Triage to RCA: Automating Critical Incident Communication with Jira, Statuspage API, Rovo, and Confluence'
description = 'A practical workflow for triaging critical incidents, publishing sanitized public updates, and generating RCA documentation with Jira, Statuspage API, Rovo, and Confluence.'
tags = ['jira', 'jira service management', 'statuspage', 'rovo', 'confluence', 'incident management', 'automation', 'rca']
categories = ['Atlassian', 'Automation', 'Incident Management']
+++

## Introduction

When a critical incident happens, fixing the technical problem is only one part of the job. The team also needs to communicate clearly, update customers at the right time, and document what happened after the incident is over.

In many teams, these steps are still disconnected. The incident is triaged in Jira, the public message is written manually, and the final RCA is prepared later, sometimes when part of the context is already lost.

In this article, I will show a practical way to connect these steps.

The flow starts in Jira Service Management, where the incident is created and triaged. If the issue is serious enough, Jira Automation opens a public incident in Statuspage. Rovo helps generate public messages in a safer way by removing sensitive internal details before the update is published. Later, when the incident is resolved, the same process sends a final public update and prepares the RCA documentation.

The goal here is simple: reduce manual work during critical incidents, make communication more consistent, and leave better documentation for future analysis.

---

## What each tool does in this flow

Before going into the automations, it is useful to explain the role of each product.

### Jira Service Management

Jira Service Management is the main place where the incident is created, triaged, and tracked. In this flow, it works as the source of truth for the incident lifecycle.

### Statuspage

Statuspage is the public communication layer. It is the place where customers or stakeholders can see incident updates, service degradation, and resolution messages. In this implementation, Jira Automation updates Statuspage through API calls.

### Rovo

Rovo is Atlassian's AI capability. In this flow, it helps with tasks that normally take manual effort, such as classifying the incident, rewriting internal notes into public-safe messages, and generating the first draft of the RCA.

### Confluence

Confluence is the documentation layer. After the incident is resolved, it is used to keep the final RCA page so the team can review what happened, what was learned, and what needs to be improved.

---

## Why this workflow matters

Incident communication is still a weak point in many teams.

Common problems include:

- Public updates are delayed because the technical team is focused on investigation.
- External messages are written in inconsistent language.
- Sensitive internal details may leak into customer-facing updates.
- The public incident timeline is not aligned with the internal Jira workflow.
- RCA documentation is written too late, when context has already been lost.

By connecting Jira, Statuspage, Rovo, and Confluence in one flow, it becomes possible to:

- reduce the time between triage and public communication
- standardize the language used in public updates
- avoid exposing internal details
- keep the public timeline aligned with the internal issue lifecycle
- generate RCA documentation at the end of the incident while the context is still fresh

---

## High-level architecture

This implementation is based on three automation rules:

1. **Automation 1 — Incident creation and triage**
   - A new incident is created in Jira.
   - Rovo classifies the incident.
   - If it is critical and requires external communication, the rule opens a public incident in Statuspage.
   - The returned Statuspage incident ID is saved in Jira.

2. **Automation 2 — Incident update**
   - The Jira incident is updated during investigation.
   - Rovo generates a new sanitized public message based on the current lifecycle stage.
   - Jira Automation updates the existing Statuspage incident by API.

3. **Automation 3 — Incident resolution and RCA generation**
   - The Jira incident is resolved.
   - Jira Automation closes the Statuspage incident with a final public message.
   - Rovo generates the RCA draft.
   - Jira Automation creates a Confluence page natively and stores the page link back in Jira.

---

## Prerequisites

Before building the automation rules, make sure the following items already exist.

### Required products

- Jira Cloud / Jira Service Management
- Statuspage
- Confluence Cloud
- Rovo / Atlassian AI enabled in your environment
- Jira Automation access with web request enabled

### Suggested custom fields in Jira

Create these fields before configuring the rules:

- **Statuspage Incident ID** — single-line text
- **Public Communication Required** — checkbox or select list
- **Incident Severity** — select list
- **Incident Criticality** — select list
- **Sanitized Public Message** — paragraph text
- **Last Public Update Stage** — select list
- **Confluence RCA URL** — URL or text field
- **RCA Draft** — paragraph text or long text field

### Suggested lifecycle values

You can model your public update stages with a field such as **Last Public Update Stage**.

Recommended values:

- investigating
- identified
- workaround_available
- fix_in_progress
- deploying_fix
- monitoring
- resolved

This makes the update automation easier to maintain.

---

# Automation 1 — Incident creation and triage

This automation is responsible for:

- detecting new incidents
- classifying them with Rovo
- deciding whether public communication is required
- opening the public incident in Statuspage
- storing the returned Statuspage incident ID in Jira

---

## Rule 1A — Classify the incident when it is created

### Rule scope

```text
Project scope: your Jira Service Management project
Rule type: project rule
```

### Trigger

```text
Trigger: Work item created
```

### Conditions

```text
Issue type equals Incident
Summary is not empty
Description is not empty
```

### Flow

1. Trigger the rule when the incident is created.
2. Confirm the issue type is Incident.
3. Use Rovo to classify the issue.
4. Store the triage result in a variable.
5. Update the issue fields with the classification.
6. Decide whether the incident should move to public communication.

### Rovo prompt for incident triage

```text
You are helping triage a newly created incident in Jira Service Management.

Your task is to analyze the incident summary and description and classify it for internal incident handling.

Return your answer in valid JSON only.

Rules:
- Do not add explanations outside the JSON.
- Use simple values only.
- If the description is unclear, make the most reasonable classification based on the available information.
- Do not invent technical details that are not present.
- "public_communication_required" must be true only if the issue appears to affect customers, external users, or business-critical services.
- "criticality" must be "critical" only when the incident suggests major business impact, major service degradation, or outage.
- "priority_name" must match one of your exact Jira priority names. In this example, use: Low, Medium, High, Highest.
- "severity" must be one of: sev4, sev3, sev2, sev1

JSON format:
{
  "priority_name": "",
  "severity": "",
  "criticality": "",
  "public_communication_required": false,
  "reason": ""
}

Input:
Issue key: {{issue.key}}
Summary: {{issue.summary}}
Description: {{issue.description}}
Reporter: {{issue.reporter.displayName}}
Service: {{issue.customfield_service}}
```

### Expected output

```json
{
  "priority_name": "Highest",
  "severity": "sev1",
  "criticality": "critical",
  "public_communication_required": true,
  "reason": "The issue appears to affect a customer-facing service and suggests a major service disruption."
}
```

### Store the triage output

```text
Action: Create variable
Variable name: triageResultRaw
Smart value:
{{agentResponse.asString}}
```

> For structured JSON parsing in later actions, use `{{agentResponse.asObject}}`. The raw string variable is only useful for audit comments or debugging.

### Optional internal audit comment

```text
Action: Add internal comment

Triage result:
{{triageResultRaw}}
```

### Update Jira fields

Example mapping:

```text
Incident Severity              <- sev1 / sev2 / sev3 / sev4
Priority                       <- Highest / High / Medium / Low
Incident Criticality           <- critical / non-critical
Public Communication Required  <- true / false
```

Example advanced JSON edit:

```json
{
  "fields": {
    "priority": {
      "name": "{{agentResponse.asObject.priority_name}}"
    },
    "customfield_incident_severity": "{{agentResponse.asObject.severity}}",
    "customfield_incident_criticality": "{{agentResponse.asObject.criticality}}",
    "customfield_public_communication_required": {{agentResponse.asObject.public_communication_required}}
  }
}
```

### Decision point

Only continue to public communication when all conditions below are true:

```text
Priority is High or Highest
Incident Criticality is Critical
Public Communication Required is true
```

---

## Rule 1B — Open the public incident in Statuspage

Once the incident is classified as critical, the next part of the rule creates the public incident.

### Conditions

```text
Issue type equals Incident
Priority equals High or Highest
Incident Criticality equals Critical
Public Communication Required equals true
Statuspage Incident ID is empty
```

### Generate the first public message with Rovo

```text
You are helping write a public Statuspage update for a newly identified critical incident.

Task:
Rewrite the internal incident details into a short, clear, customer-facing message.

Rules:
- Do not mention customer names.
- Do not mention internal ticket IDs, account IDs, internal URLs, hostnames, pod names, region names, stack traces, or IP addresses.
- Do not expose internal team names or escalation details.
- Do not speculate about the root cause.
- Do not assign blame.
- Use calm and professional language.
- Mention only the affected service or capability in a generic way.
- Mention that the team is investigating.
- Keep the output between 2 and 4 sentences.
- Output only the final message.

Input:
Issue key: {{issue.key}}
Summary: {{issue.summary}}
Description: {{issue.description}}
Latest internal comment: {{issue.comments.last.body}}
Severity: {{issue.customfield_incident_severity}}
Service: {{issue.customfield_service}}
```

### Store the public message

```text
Action: Create variable
Variable name: publicMessage
Smart value:
{{agentResponse.asString}}
```

> `{{agentResponse.asString}}` is safer here because the message will be passed to an external API as plain text.

### Optional traceability field

```text
Action: Edit issue
Field: Sanitized Public Message
Value: {{publicMessage}}
```

### Create the Statuspage incident

```http
POST https://api.statuspage.io/v1/pages/{page_id}/incidents
Authorization: OAuth {STATUSPAGE_API_KEY}
Content-Type: application/json
```

### Request body

```json
{
  "incident": {
    "name": "{{issue.key}} - {{issue.summary}}",
    "status": "investigating",
    "impact_override": "critical",
    "deliver_notifications": true,
    "body": "{{publicMessage}}",
    "metadata": {
      "jira_issue_key": "{{issue.key}}"
    }
  }
}
```

### Jira Automation web request

```text
Action: Send web request
Method: POST
URL: https://api.statuspage.io/v1/pages/{page_id}/incidents
Headers:
  Authorization: OAuth {STATUSPAGE_API_KEY}
  Content-Type: application/json
Web request body: custom data
Wait for response: enabled
```

### Save the returned Statuspage incident ID

```text
Action: Create variable
Variable name: statuspageIncidentId
Smart value:
{{webResponse.body.id}}
```

```text
Action: Edit issue
Field: Statuspage Incident ID
Value: {{statuspageIncidentId}}
```

### Optional internal comment

```text
Action: Add internal comment

Statuspage incident created successfully.
Statuspage Incident ID: {{statuspageIncidentId}}
Public message: {{publicMessage}}
```

---

# Automation 2 — Incident update and public communication

This automation keeps the public incident aligned with the internal investigation lifecycle.

Instead of using only one generic update message, this rule uses the current incident stage to produce a more accurate public update.

---

## Purpose of this rule

This rule should run whenever the Jira incident changes in a meaningful way, such as:

- incident stage changed
- new internal investigation update added
- service degradation confirmed
- workaround available
- fix is being prepared
- deployment started
- validation in progress
- monitoring phase started

---

## Trigger

```text
Trigger: Work item updated
```

### Recommended conditions

```text
Issue type equals Incident
Statuspage Incident ID is not empty
Public Communication Required equals true
Incident Criticality equals Critical
```

### Recommended extra condition

To avoid noise, only run the rule if one of these changed:

```text
Incident stage field changed
Latest internal note field changed
Status changed
Resolution field changed
```

If your process uses a dedicated field such as **Last Public Update Stage**, that is even better.

---

## Suggested public stages

Use a field that represents the public communication stage. Example values:

```text
investigating
identified
workaround_available
fix_in_progress
deploying_fix
monitoring
```

This gives Rovo context to generate the right type of update.

---

## Create a variable for the current stage

```text
Action: Create variable
Variable name: incidentStage
Smart value:
{{issue.customfield_last_public_update_stage}}
```

---

## Rovo prompt for update messages

```text
You are helping write a public Statuspage update for an active critical incident.

Task:
Generate a short customer-facing update based on the current incident stage and the latest internal information.

Rules:
- Do not mention customer names.
- Do not mention internal ticket IDs, account IDs, hostnames, internal URLs, IP addresses, stack traces, database names, pod names, secret values, or internal-only deployment details.
- Do not expose internal team names or escalation details.
- Do not speculate beyond the confirmed state.
- Do not assign blame.
- Use calm and professional language.
- Keep the output between 2 and 4 sentences.
- Output only the final message.
- Adapt the tone and wording based on the stage:
  - investigating: say the team is investigating
  - identified: say the team identified the issue and is working on mitigation
  - workaround_available: say a workaround is available for some users when confirmed
  - fix_in_progress: say a fix is being prepared or applied
  - deploying_fix: say corrective changes are being deployed carefully
  - monitoring: say the service is recovering and is being monitored closely

Input:
Issue key: {{issue.key}}
Summary: {{issue.summary}}
Current stage: {{incidentStage}}
Latest internal comment: {{issue.comments.last.body}}
Description: {{issue.description}}
Service: {{issue.customfield_service}}
Severity: {{issue.customfield_incident_severity}}
```

---

## Example Rovo outputs by stage

### Investigating

```text
We are currently investigating an issue affecting part of the service experience for some users. Our team is actively reviewing the situation and working to identify the cause. We will share another update as soon as more information is confirmed.
```

### Identified

```text
We have identified the cause of the current service disruption and are working on mitigation actions. Our team is focused on restoring normal behavior as quickly as possible. We will continue to share updates as progress is confirmed.
```

### Fix in progress

```text
We are currently applying corrective changes related to the ongoing service issue. Our team is working carefully to restore normal service while minimizing further impact. Another update will be shared as soon as validation progresses.
```

### Deploying fix

```text
We are deploying a corrective change to address the current issue. The team is monitoring the rollout closely to confirm service recovery. We will provide another update once validation is complete.
```

### Monitoring

```text
Service performance has improved and we are now monitoring the environment closely. The team is validating stability to confirm that the issue has been fully mitigated. We will continue to share updates until recovery is fully confirmed.
```

---

## Store the update message

```text
Action: Create variable
Variable name: publicUpdateMessage
Smart value:
{{agentResponse.asString}}
```

Optional:

```text
Action: Edit issue
Field: Sanitized Public Message
Value: {{publicUpdateMessage}}
```

---

## Map Jira stages to Statuspage statuses

You can map the internal stage to the Statuspage incident status before calling the API.

Suggested mapping:

```text
investigating        -> investigating
identified           -> identified
workaround_available -> identified
fix_in_progress      -> identified
deploying_fix        -> identified
monitoring           -> monitoring
```

If you want cleaner logic, store the mapped value in a variable.

```text
Action: Create variable
Variable name: statuspageStatus
Smart value:
{{#if(equals(incidentStage,"monitoring"))}}monitoring{{else}}identified{{/}}
```

If your automation setup does not support this exact expression, replace it with conditional branches.

---

## Update the existing Statuspage incident

```http
PATCH https://api.statuspage.io/v1/pages/{page_id}/incidents/{{issue.customfield_statuspage_incident_id}}
Authorization: OAuth {STATUSPAGE_API_KEY}
Content-Type: application/json
```

### Request body

```json
{
  "incident": {
    "status": "{{statuspageStatus}}",
    "deliver_notifications": true,
    "body": "{{publicUpdateMessage}}"
  }
}
```

### Jira Automation web request

```text
Action: Send web request
Method: PATCH
URL: https://api.statuspage.io/v1/pages/{page_id}/incidents/{{issue.customfield_statuspage_incident_id}}
Headers:
  Authorization: OAuth {STATUSPAGE_API_KEY}
  Content-Type: application/json
Web request body: custom data
Wait for response: enabled
```

### Optional internal comment

```text
Action: Add internal comment

Statuspage incident updated successfully.
Stage: {{incidentStage}}
Public update: {{publicUpdateMessage}}
```

---

## Notes for production use

To avoid excessive public notifications, it is a good idea to add one or more of the controls below:

- only notify when the public stage changes
- suppress duplicate updates if the message did not change
- add a minimum interval between public notifications
- store the last published public stage in a field and compare before sending

This keeps the public timeline clean and avoids over-communication.

---

# Automation 3 — Incident resolution and RCA registration

This automation closes the public communication loop and prepares the final incident documentation.

It is responsible for:

- generating the final public resolution message
- resolving the Statuspage incident
- generating the RCA draft with Rovo
- storing the RCA draft in Jira
- creating a Confluence page natively
- saving the Confluence page URL back in Jira

---

## Trigger

```text
Trigger: Work item transitioned to Resolved
```

You can also use:

```text
Trigger: Work item updated
Condition: Resolution is not empty
```

---

## Conditions

```text
Issue type equals Incident
Statuspage Incident ID is not empty
Public Communication Required equals true
Incident Criticality equals Critical
```

---

## Generate the final customer-facing resolution message

```text
You are helping write the final public Statuspage update for a resolved critical incident.

Task:
Turn the internal resolution notes into a concise and safe customer-facing message.

Rules:
- Do not mention customer names.
- Do not mention internal ticket IDs, account IDs, IP addresses, hostnames, internal URLs, database names, pod names, secret values, stack traces, or internal-only deployment details.
- Do not expose internal team names or escalation details.
- Do not speculate if the root cause is still under investigation.
- Clearly state that the issue has been resolved.
- Briefly describe the customer impact in generic terms.
- Briefly describe the resolution in safe external language.
- Say that monitoring may continue if appropriate.
- Keep the output between 2 and 4 sentences.
- Output only the final message.

Input:
Issue key: {{issue.key}}
Summary: {{issue.summary}}
Resolution notes: {{issue.comments.last.body}}
Description: {{issue.description}}
Service: {{issue.customfield_service}}
Severity: {{issue.customfield_incident_severity}}
```

### Store the final public message

```text
Action: Create variable
Variable name: finalPublicMessage
Smart value:
{{agentResponse.asString}}
```

---

## Resolve the Statuspage incident

```http
PATCH https://api.statuspage.io/v1/pages/{page_id}/incidents/{{issue.customfield_statuspage_incident_id}}
Authorization: OAuth {STATUSPAGE_API_KEY}
Content-Type: application/json
```

### Request body

```json
{
  "incident": {
    "status": "resolved",
    "deliver_notifications": true,
    "body": "{{finalPublicMessage}}"
  }
}
```

### Jira Automation web request

```text
Action: Send web request
Method: PATCH
URL: https://api.statuspage.io/v1/pages/{page_id}/incidents/{{issue.customfield_statuspage_incident_id}}
Headers:
  Authorization: OAuth {STATUSPAGE_API_KEY}
  Content-Type: application/json
Web request body: custom data
Wait for response: enabled
```

### Optional internal comment

```text
Action: Add internal comment

Statuspage incident resolved successfully.
Final public message: {{finalPublicMessage}}
```

---

## Generate the RCA draft with Rovo

Once the public incident is closed, the workflow can generate the RCA draft.

### Rovo prompt for RCA generation

```text
You are helping generate an RCA draft for a resolved critical incident.

Task:
Create a structured RCA draft in professional English.

Output format:
Use the following sections exactly:

1. Executive summary
2. Customer impact
3. Detection
4. Timeline
5. Root cause
6. Contributing factors
7. Resolution
8. Preventive and corrective actions
9. Lessons learned
10. Follow-up actions

Rules:
- If the root cause is not fully confirmed, explicitly say that it is still under investigation.
- Do not include customer names, account IDs, IP addresses, internal URLs, credentials, secrets, or private infrastructure identifiers.
- Use a clear and objective tone.
- Keep the timeline chronological.
- Convert action items into bullet points.
- Output only the RCA draft.

Input:
Issue key: {{issue.key}}
Summary: {{issue.summary}}
Description: {{issue.description}}
Created date: {{issue.created}}
Resolved date: {{issue.resolutiondate}}
Latest internal comments: {{issue.comments}}
Resolution notes: {{issue.comments.last.body}}
Severity: {{issue.customfield_incident_severity}}
Service: {{issue.customfield_service}}
```

### Store the RCA draft

```text
Action: Create variable
Variable name: rcaDraft
Smart value:
{{agentResponse.asString}}
```

### Save the RCA draft in Jira

```text
Action: Edit issue
Field: RCA Draft
Value: {{rcaDraft}}
```

Optional:

```text
Action: Add internal comment

RCA draft generated successfully.

{{rcaDraft}}
```

---

## Create the Confluence page natively

At this point, Jira Automation can create the Confluence page using the native **Create Confluence page** action.

This is the cleanest approach when the main goal is to create the page and keep the process native inside Atlassian Automation.

### Jira Automation action

```text
Action: Create Confluence page
Site: your connected Confluence site
Space: your target RCA space
Parent page: optional
Content type: page
Title:
RCA - {{issue.key}} - {{issue.summary}}
```

### Available smart values after page creation

```text
{{createdPage.title}}
{{createdPage.url}}
```

### Save the Confluence page URL back in Jira

```text
Action: Edit issue
Field: Confluence RCA URL
Value: {{createdPage.url}}
```

### Optional internal comment

```text
Action: Add internal comment

Confluence RCA page created successfully.
Page title: {{createdPage.title}}
Page URL: {{createdPage.url}}
```

---

## Important note about the native Confluence action

The native **Create Confluence page** action is the best option when you want to create the page without an extra API call. However, it does not let you populate the full page body during creation.

Because of that, this version of the flow works like this:

- Rovo generates the RCA draft
- Jira stores the RCA draft
- Jira Automation creates the destination page in Confluence
- the team can review and move the draft content into the final page

If you need the Confluence page body to be filled automatically at creation time, that becomes an advanced variant and usually requires the Confluence API instead of the native action.

---

# Final checklist

Before moving this solution to production, validate the following points:

- [ ] The rule exits safely if triggered twice.
- [ ] Statuspage Incident ID is only created once.
- [ ] Update automation does not publish duplicate public messages.
- [ ] Sensitive details are never exposed by the prompts.
- [ ] Jira custom fields are mapped correctly.
- [ ] Statuspage API credentials are stored securely.
- [ ] Public lifecycle stages are standardized.
- [ ] RCA generation works even when the root cause is not fully confirmed.
- [ ] The Confluence page is created in the correct space.

---

## Final thoughts

The real value of this flow is not only automation. The main gain is consistency: triage starts communication, public updates follow the real incident lifecycle, and RCA documentation becomes part of the process instead of an afterthought.

By connecting Jira, Statuspage, Rovo, and Confluence, teams can handle critical incidents in a more organized way and leave better documentation for the next similar case.
