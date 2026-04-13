+++
date = '2026-04-13T10:14:47-03:00'
draft = true
title = 'From Triage to RCA: Automating Critical Incident Communication with Jira, Statuspage API, Rovo, and Confluence'
description = 'A practical workflow that connects incident triage, public communication, and RCA generation using Jira, Statuspage API, Rovo, and Confluence.'
tags = ['jira', 'jira service management', 'statuspage', 'rovo', 'confluence', 'incident management', 'automation', 'rca']
categories = ['Atlassian', 'Automation', 'Incident Management']
+++

## Introduction

Critical incidents need more than a fast technical response. They also need clear communication, consistent public updates, and a structured post-incident process. In many teams, these steps still happen in separate ways: triage is done in Jira, customer communication is written manually, and the RCA is created later with little automation.

In this article, I will present a practical workflow that connects these stages. The process starts with incident triage in Jira, where Rovo can also help support the analysis and identify whether the issue should be treated as a critical, high-priority incident. After that, Jira automation can publish updates automatically to Statuspage through API calls, using Rovo to generate customer-facing messages with sensitive information removed.

When the incident is resolved, the same flow can update and close the public incident in Statuspage with a final message also supported by Rovo. After that, the process can generate a structured RCA draft in Markdown, ready to be published in Confluence.

RCA stands for Root Cause Analysis. It is the document used to explain what happened, what the customer impact was, what caused the incident, how the team resolved it, and what actions should be taken to reduce the chance of the same issue happening again.

The goal is not only to automate tasks, but also to make incident management faster, more reliable, and more repeatable. At the same time, this process helps create better documentation for future consultation, which can also support Rovo in future triage and incident analysis.

## Why this workflow matters

Incident communication is still a weak point in many teams. In many cases, the investigation happens internally, but external communication is delayed, inconsistent, or too dependent on manual work. This increases the risk of noise, rework, and even the accidental exposure of sensitive information.

By connecting Jira, Statuspage, Rovo, and Confluence in one workflow, it becomes possible to reduce the time between triage and public communication, standardize the language used in updates, avoid exposing internal details, and make sure the incident learning is recorded in a structured way at the end of the process.

## High-level architecture

This workflow follows five main steps:

1. The incident is triaged in Jira.
2. Rovo generates a sanitized external message.
3. Jira automation opens the incident in Statuspage through API.
4. When the incident is resolved, Jira automation updates and closes the public incident in Statuspage with a Rovo-generated message.
5. An RCA page is created automatically in Confluence.

## Implementation model

In practice, I recommend implementing this workflow through **three automation rules**:

1. **Creation automation**  
   Triggered when the incident is created. It performs triage, decides whether public communication is required, generates the first public message, opens the Statuspage incident, and stores the returned Statuspage incident ID.

2. **Update automation**  
   Triggered when the incident is updated. It evaluates the current incident stage, generates a new public update message with Rovo, and updates the existing Statuspage incident with the proper external status.

3. **Finalization automation**  
   Triggered when the incident is resolved. It closes the public incident in Statuspage, generates the RCA content with Rovo, creates the Confluence page through API, and stores the Confluence link back in Jira.

---

## Step 1 — Incident creation and triage in Jira

The first step of the workflow starts when a new incident is created in Jira Service Management. At this stage, the goal is to classify the incident as early as possible, decide whether it requires public communication, and prepare the data that will be used by the next automation steps.

To keep the process maintainable, I recommend splitting this step into two logical blocks inside the same creation automation:

- **Rule block 1A — Incident created and triaged**
- **Rule block 1B — Publish to Statuspage if the incident is critical**

Before creating the rules, make sure the following fields already exist in Jira:

- **Statuspage Incident ID** — single-line text field
- **Public Communication Required** — checkbox or select list
- **Incident Severity** — select list
- **Incident Criticality** — select list or checkbox
- **Sanitized Public Message** — paragraph text field (optional, but useful for auditability)
- **Incident Progress Stage** — select list
- **RCA Page URL** — URL field
- **Last Public Update Message** — paragraph text field (optional)

---

## Rule block 1A — Incident created and triaged

This part starts when a new incident is created. It runs inside the Jira Service Management project scope and performs the first classification step.

### Rule scope

```text
Project scope: your Jira Service Management project
Rule type: project rule
```

### Trigger

```text
Trigger: Work item created
```

### Recommended conditions

```text
Issue type equals Incident
Summary is not empty
Description is not empty
```

### Suggested flow

1. Trigger when the incident is created
2. Confirm the issue type is Incident
3. Use Rovo to classify the incident
4. Store the triage output
5. Update Jira fields with the classification result
6. Decide whether the issue should move to public communication

### Suggested Rovo prompt for triage

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
- "priority" must be one of: low, medium, high, highest
- "severity" must be one of: sev4, sev3, sev2, sev1
- "progress_stage" must be one of: analyzing, investigating, identified, fixing, preparing_deploy, deploying_fix, validating_fix, monitoring, resolved

JSON format:
{
  "priority": "",
  "severity": "",
  "criticality": "",
  "public_communication_required": false,
  "progress_stage": "",
  "reason": ""
}

Input:
Issue key: {{issue.key}}
Summary: {{issue.summary}}
Description: {{issue.description}}
Reporter: {{issue.reporter.displayName}}
Service: {{issue.customfield_service}}
```

### Expected Rovo output

```json
{
  "priority": "highest",
  "severity": "sev1",
  "criticality": "critical",
  "public_communication_required": true,
  "progress_stage": "investigating",
  "reason": "The issue appears to affect a customer-facing service and suggests a major service disruption."
}
```

### Store the triage result

After the **Use Rovo** action, store the result in a variable so it can be reused later in the rule.

```text
Action: Create variable
Variable name: triageResult
Smart value:
{{rovo.response}}
```

If you prefer, you can keep the raw JSON response in an internal comment for easier troubleshooting.

```text
Action: Add internal comment

Triage result:
{{triageResult}}
```

### Update the issue fields

After the triage output is available, update the issue with the values that will drive the rest of the workflow.

Example mapping:

```text
Incident Severity              <- sev1 / sev2 / sev3 / sev4
Priority                       <- highest / high / medium / low
Incident Criticality           <- critical / non-critical
Public Communication Required  <- true / false
Incident Progress Stage        <- analyzing / investigating / identified / fixing / preparing_deploy / deploying_fix / validating_fix / monitoring / resolved
```

If you use advanced field editing, the structure can look like this:

```json
{
  "fields": {
    "priority": {
      "name": "Highest"
    },
    "customfield_incident_severity": "sev1",
    "customfield_incident_criticality": "critical",
    "customfield_public_communication_required": true,
    "customfield_incident_progress_stage": "investigating"
  }
}
```

### Decision point

At the end of Rule block 1A, the issue has already been triaged. The next step is to decide whether it should be published externally.

Recommended logic:

```text
If:
- Priority is High or Highest
- Incident Criticality is Critical
- Public Communication Required is true

Then:
- Continue to public communication flow
Else:
- Stop the rule
```

---

## Rule block 1B — Publish to Statuspage if the incident is critical

Once the incident is classified as high-impact and critical, the second part of the creation automation prepares the external message and opens the public incident in Statuspage.

### Trigger

```text
Trigger: Work item created
```

### Conditions

```text
Issue type equals Incident
Priority equals High or Highest
Incident Criticality equals Critical
Public Communication Required equals true
Statuspage Incident ID is empty
```

### Generate the public message with Rovo

At this point, the internal incident details must be transformed into a safe public message.

Use Rovo with a prompt like this:

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
- Mention only the affected service or customer-facing capability in a generic way.
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
Progress stage: {{issue.customfield_incident_progress_stage}}
```

### Store the public message

Store the sanitized response in a variable first.

```text
Action: Create variable
Variable name: publicMessage
Smart value:
{{rovo.response}}
```

Optionally, also save it into a Jira field for traceability.

```text
Action: Edit issue
Field: Sanitized Public Message
Value: {{publicMessage}}
```

### Create the Statuspage incident

Now the rule can call the Statuspage API and open the public incident.

```http
POST https://api.statuspage.io/v1/pages/{page_id}/incidents
Authorization: OAuth {STATUSPAGE_API_KEY}
Content-Type: application/json
```

### Example request body

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

### Jira Automation web request configuration

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

After the API call, Jira Automation can store the response body in a variable and extract the returned incident ID.

Example:

```text
Action: Create variable
Variable name: statuspageIncidentId
Smart value:
{{webResponse.body.id}}
```

Then write that value back to the Jira issue.

```text
Action: Edit issue
Field: Statuspage Incident ID
Value: {{statuspageIncidentId}}
```

This step is very important because the same ID will be used later to update and resolve the public incident.

### Optional internal comment for traceability

```text
Action: Add internal comment

Statuspage incident created successfully.
Statuspage Incident ID: {{statuspageIncidentId}}
Public message: {{publicMessage}}
```

### Summary of the creation automation

At the end of the first automation, the workflow already does four important things:

1. It identifies and triages the incident in Jira.
2. It classifies whether the issue requires public communication.
3. It generates a sanitized public message with Rovo.
4. It opens the public incident in Statuspage and stores the returned incident ID in Jira.

This creates the foundation for the next automation, where the incident can be updated during the investigation.

---

## Step 2 — Incident updates and public progress communication

Once the public incident already exists in Statuspage, the next automation keeps the public timeline updated while the technical investigation continues. This rule is responsible for reacting to meaningful internal progress such as investigation updates, issue identification, workaround progress, deployment preparation, deployment execution, validation, and monitoring.

The important design principle here is simple: not every internal change should produce a public update. The automation should react only to relevant milestones and translate them into the correct external communication state.

### Additional field recommendations

Before implementing this rule, I recommend the following field strategy:

- **Incident Progress Stage** — select list  
  Suggested values:
  - analyzing
  - investigating
  - identified
  - fixing
  - preparing_deploy
  - deploying_fix
  - validating_fix
  - monitoring
  - resolved

- **Last Public Update Message** — paragraph text field (optional)
- **Last Public Update Timestamp** — date time field (optional)
- **Public Update Required** — checkbox (optional, useful when you want stricter control)

### Automation 2 — Update the existing Statuspage incident

This automation runs when the Jira incident is updated after the initial creation flow.

### Rule scope

```text
Project scope: your Jira Service Management project
Rule type: project rule
```

### Trigger

```text
Trigger: Work item updated
```

### Recommended trigger filters

To avoid too many executions, restrict the rule to meaningful changes only. A practical approach is to run it when one of the following changes:

```text
Incident Progress Stage changed
Status changed
Fix version changed
Internal comment added
Public Update Required changed to true
```

### Recommended conditions

```text
Issue type equals Incident
Statuspage Incident ID is not empty
Public Communication Required equals true
Resolution is empty
```

### Loop protection

Because this rule edits the issue and calls external APIs, it is a good idea to add loop protection.

Recommended options:

```text
Only continue if the actor is not Automation for Jira
OR
Only continue if Incident Progress Stage changed
OR
Only continue if Public Update Required equals true
```

### Internal-to-external status mapping

Statuspage public statuses are usually simpler than the internal operational stages. Because of that, I recommend using a mapping layer inside the automation.

Suggested mapping:

```text
analyzing, investigating                  -> investigating
identified, fixing                        -> identified
preparing_deploy, deploying_fix           -> identified
validating_fix, monitoring                -> monitoring
resolved                                  -> handled by the finalization automation
```

### Suggested rule flow

1. Trigger when the incident is updated
2. Confirm that a Statuspage incident already exists
3. Read the current Incident Progress Stage
4. Map the internal stage to the public Statuspage status
5. Use Rovo to generate a customer-facing progress update
6. Send a PATCH request to Statuspage
7. Store the published message in Jira
8. Optionally reset `Public Update Required` to false

### Example logic for status mapping

You can implement the mapping with **If / else blocks** inside Jira Automation.

```text
IF Incident Progress Stage is analyzing or investigating
  statuspageStatus = investigating

ELSE IF Incident Progress Stage is identified or fixing
  statuspageStatus = identified

ELSE IF Incident Progress Stage is preparing_deploy or deploying_fix
  statuspageStatus = identified

ELSE IF Incident Progress Stage is validating_fix or monitoring
  statuspageStatus = monitoring

ELSE
  Stop the rule
```

### Example variable creation

```text
Action: Create variable
Variable name: statuspageStatus
Value: identified
```

In a real rule, the value above would be assigned dynamically through branches.

### Suggested Rovo prompt for progress updates

Once the public status is known, the next step is generating a safe customer-facing message.

```text
You are helping write a public Statuspage progress update for an active incident.

Task:
Write a short and professional customer-facing update based on the current internal incident stage.

Rules:
- Do not mention customer names.
- Do not mention account IDs, internal ticket IDs, internal URLs, pod names, region names, stack traces, IP addresses, or hostnames.
- Do not expose security-sensitive details.
- Do not speculate about the root cause unless it is already confirmed.
- Use the current progress stage to guide the wording.
- Keep the tone calm and professional.
- Keep the update between 2 and 4 sentences.
- The message must be understandable by external users.
- Output only the final message.

Stage guidance:
- If stage is investigating or analyzing, explain that the team is still investigating.
- If stage is identified, explain that the issue has been identified and the team is working on mitigation.
- If stage is fixing, explain that corrective actions are in progress.
- If stage is preparing_deploy, explain that a fix is being prepared for deployment.
- If stage is deploying_fix, explain that changes are being applied carefully.
- If stage is validating_fix, explain that the team is validating the recovery.
- If stage is monitoring, explain that service is recovering and is being monitored.

Input:
Issue key: {{issue.key}}
Summary: {{issue.summary}}
Description: {{issue.description}}
Current stage: {{issue.customfield_incident_progress_stage}}
Latest internal note: {{issue.comments.last.body}}
Service: {{issue.customfield_service}}
Severity: {{issue.customfield_incident_severity}}
```

### Example generated outputs

**Example for `preparing_deploy`:**

```text
We have identified the issue affecting the service and are preparing a corrective change for deployment. Our team is proceeding carefully to restore normal behavior as quickly as possible.
```

**Example for `deploying_fix`:**

```text
We have identified the issue and are currently applying corrective changes. We are monitoring the deployment process closely before confirming recovery.
```

**Example for `validating_fix`:**

```text
Corrective changes have been applied and our team is now validating service recovery. We will provide another update once validation is complete.
```

**Example for `monitoring`:**

```text
Service behavior has improved and we are currently monitoring the platform closely. We will continue to observe stability before marking the incident as resolved.
```

### Store the generated progress message

```text
Action: Create variable
Variable name: publicUpdateMessage
Smart value:
{{rovo.response}}
```

You can also store it in Jira:

```text
Action: Edit issue
Field: Last Public Update Message
Value: {{publicUpdateMessage}}
```

### Update the incident in Statuspage

Use the Statuspage incident ID stored in Jira to update the existing public incident.

```http
PATCH https://api.statuspage.io/v1/pages/{page_id}/incidents/{incident_id}
Authorization: OAuth {STATUSPAGE_API_KEY}
Content-Type: application/json
```

### Example request body

```json
{
  "incident": {
    "status": "{{statuspageStatus}}",
    "body": "{{publicUpdateMessage}}",
    "deliver_notifications": true
  }
}
```

### Optional component-level update

If your Statuspage setup uses components, this is also the moment to degrade or improve their public state.

Example idea:

```json
{
  "incident": {
    "status": "{{statuspageStatus}}",
    "body": "{{publicUpdateMessage}}",
    "deliver_notifications": true,
    "components": {
      "component_id_1": "degraded_performance",
      "component_id_2": "partial_outage"
    }
  }
}
```

Use this only if your team has already defined a reliable mapping between Jira services and Statuspage component IDs.

### Jira Automation web request configuration

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

### Save update metadata back in Jira

After a successful API call, I recommend saving the last published update and timestamp.

```text
Action: Edit issue
Field: Last Public Update Message
Value: {{publicUpdateMessage}}
```

```text
Action: Edit issue
Field: Last Public Update Timestamp
Value: {{now}}
```

If you use a manual checkbox to force updates, reset it after publication.

```text
Action: Edit issue
Field: Public Update Required
Value: false
```

### Optional internal audit comment

```text
Action: Add internal comment

Statuspage incident updated successfully.
Statuspage Incident ID: {{issue.customfield_statuspage_incident_id}}
Public status: {{statuspageStatus}}
Published message: {{publicUpdateMessage}}
```

### Summary of the update automation

At the end of the second automation, the workflow does three important things:

1. It watches the internal progress of the incident.
2. It translates internal technical stages into safe external communication.
3. It keeps the public Statuspage timeline updated without creating a second incident.

This makes the public communication flow consistent during the entire active lifecycle of the incident.

---

## Step 3 — Incident finalization, RCA generation, and Confluence publication

The final automation begins when the incident is resolved internally. At this moment, the workflow must close the public Statuspage incident, generate the final RCA content, and create the Confluence page that will serve as the post-incident record.

This is the step that closes the loop from operational response to organizational learning.

### Automation 3 — Finalize the incident and create the RCA page

### Rule scope

```text
Project scope: your Jira Service Management project
Rule type: project rule
```

### Trigger

```text
Trigger: Work item transitioned
From: any active incident status
To: Resolved
```

An alternative trigger is:

```text
Trigger: Field value changed
Field: Resolution
```

### Recommended conditions

```text
Issue type equals Incident
Statuspage Incident ID is not empty
Public Communication Required equals true
RCA Page URL is empty
```

### Suggested flow

1. Trigger when the incident is resolved
2. Generate the final customer-facing resolution message with Rovo
3. Resolve the incident in Statuspage
4. Generate the RCA draft in Markdown with Rovo
5. Convert the RCA into Confluence storage format
6. Create the Confluence page through API
7. Store the Confluence page URL in Jira
8. Add an internal audit comment

### Generate the final public resolution message

Use Rovo to turn the internal resolution notes into a safe external closure message.

```text
You are helping write the final public Statuspage update for a resolved incident.

Task:
Write a short, professional customer-facing resolution message.

Rules:
- Do not mention customer names.
- Do not mention account IDs, internal URLs, hostnames, stack traces, IP addresses, database names, secrets, or private infrastructure details.
- Do not expose internal-only workaround steps.
- Do not speculate about the root cause if it is not confirmed.
- Clearly say that the incident has been resolved.
- Briefly summarize the customer-facing impact in generic language.
- Briefly summarize the resolution in safe external language.
- Keep the output between 2 and 4 sentences.
- Output only the final message.

Input:
Issue key: {{issue.key}}
Summary: {{issue.summary}}
Description: {{issue.description}}
Resolution notes: {{issue.comments.last.body}}
Service: {{issue.customfield_service}}
Severity: {{issue.customfield_incident_severity}}
```

### Store the final resolution message

```text
Action: Create variable
Variable name: finalPublicMessage
Smart value:
{{rovo.response}}
```

### Resolve the Statuspage incident

Use the same incident ID stored during the creation automation.

```http
PATCH https://api.statuspage.io/v1/pages/{page_id}/incidents/{incident_id}
Authorization: OAuth {STATUSPAGE_API_KEY}
Content-Type: application/json
```

### Example request body

```json
{
  "incident": {
    "status": "resolved",
    "body": "{{finalPublicMessage}}",
    "deliver_notifications": true
  }
}
```

### Optional component recovery update

If your page uses components, this is the right moment to return them to an operational state.

```json
{
  "incident": {
    "status": "resolved",
    "body": "{{finalPublicMessage}}",
    "deliver_notifications": true,
    "components": {
      "component_id_1": "operational",
      "component_id_2": "operational"
    }
  }
}
```

### Jira Automation web request configuration

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

### Generate the RCA in Markdown with Rovo

After the public incident is closed, generate the RCA content.

```text
You are helping generate a technical RCA draft in Markdown for a resolved critical incident.

Task:
Create a structured RCA document in professional English.

Output format:
Use the following sections exactly:
# Executive summary
# Customer impact
# Detection
# Timeline
# Triage and investigation
# Confirmed root cause
# Contributing factors
# Resolution
# Preventive and corrective actions
# Follow-up owners and target dates
# Lessons learned

Rules:
- If the root cause is not fully confirmed, write "Root cause still under investigation" instead of inventing one.
- Do not include customer names, account IDs, internal-only URLs, IP addresses, hostnames, credentials, secrets, or security-sensitive data.
- Convert informal internal notes into a professional RCA tone.
- Keep the timeline chronological and objective.
- Use bullet points where useful.
- If owners or dates are not explicitly present in the input, write "To be assigned".
- Output only the Markdown body.

Input:
Issue key: {{issue.key}}
Summary: {{issue.summary}}
Description: {{issue.description}}
Incident created: {{issue.created}}
Incident resolved: {{issue.resolutiondate}}
Severity: {{issue.customfield_incident_severity}}
Service: {{issue.customfield_service}}
All comments: {{issue.comments}}
```

### Store the RCA Markdown

```text
Action: Create variable
Variable name: rcaMarkdown
Smart value:
{{rovo.response}}
```

For auditability, you can also write it to an internal comment or a long text custom field.

```text
Action: Add internal comment

Generated RCA Markdown:
{{rcaMarkdown}}
```

### Convert the RCA to Confluence storage format

Confluence page creation through API works better with storage-format HTML than with raw Markdown. Because of that, I recommend a second Rovo step that converts the Markdown draft into safe Confluence storage body content.

```text
You are converting an incident RCA written in Markdown into Confluence storage body content.

Task:
Convert the Markdown below into clean HTML that is safe to send as a Confluence storage body.

Rules:
- Use only simple HTML tags such as h1, h2, p, ul, li, strong, em, table, tr, td, th, pre, and code.
- Do not wrap the answer in markdown fences.
- Do not add html, body, or head tags.
- Preserve the RCA section order.
- Preserve bullet points.
- Output only the HTML body.

Markdown input:
{{rcaMarkdown}}
```

### Store the Confluence storage body

```text
Action: Create variable
Variable name: confluenceBody
Smart value:
{{rovo.response}}
```

### Create the Confluence page

Now create the RCA page in Confluence through the REST API.

```http
POST https://your-domain.atlassian.net/wiki/api/v2/pages
Authorization: Basic {BASE64_EMAIL_AND_API_TOKEN}
Accept: application/json
Content-Type: application/json
```

### Example request body

```json
{
  "spaceId": "123456",
  "status": "current",
  "title": "RCA - {{issue.key}} - {{issue.summary}}",
  "parentId": "789012",
  "body": {
    "representation": "storage",
    "value": "{{confluenceBody}}"
  }
}
```

### Jira Automation web request configuration

```text
Action: Send web request
Method: POST
URL: https://your-domain.atlassian.net/wiki/api/v2/pages
Headers:
  Authorization: Basic {BASE64_EMAIL_AND_API_TOKEN}
  Accept: application/json
  Content-Type: application/json
Web request body: custom data
Wait for response: enabled
```

### Save the returned Confluence page URL

Once the page is created, save the returned page link or page ID into the Jira issue.

Example:

```text
Action: Create variable
Variable name: confluencePageId
Smart value:
{{webResponse.body.id}}
```

Then build a user-friendly URL.

```text
Action: Create variable
Variable name: confluencePageUrl
Smart value:
https://your-domain.atlassian.net/wiki/spaces/YOURSPACE/pages/{{confluencePageId}}
```

Write it back to the issue:

```text
Action: Edit issue
Field: RCA Page URL
Value: {{confluencePageUrl}}
```

### Optional internal audit comment

```text
Action: Add internal comment

Incident finalized successfully.
Statuspage Incident ID: {{issue.customfield_statuspage_incident_id}}
Final public message: {{finalPublicMessage}}
RCA page: {{confluencePageUrl}}
```

### Summary of the finalization automation

At the end of the third automation, the workflow has fully closed the incident lifecycle:

1. The public incident has been resolved in Statuspage.
2. The final customer-facing closure message has been published.
3. The RCA draft has been generated in Markdown.
4. The RCA has been converted to Confluence storage content.
5. The Confluence page has been created automatically.
6. The final documentation link has been stored back in Jira.

This is the point where the workflow moves from incident response to post-incident learning.

---

## AI guardrails and sensitive data handling

The use of AI in this workflow needs clear rules. AI should help rewrite, summarize, and organize information, but it should not invent causes, speculate about impact, or expose sensitive data. Especially in critical incidents, public updates must reflect only confirmed facts.

For this reason, the use of Rovo should be guided by prompts that reinforce the removal of internal data, neutral language, and the prohibition of unsupported conclusions. The same care applies to the RCA, which can be more detailed but still should avoid restricted or unnecessarily sensitive information.

A practical policy is:

- AI can rewrite internal notes into customer-safe language.
- AI cannot decide root cause certainty by itself.
- AI cannot expose secrets, identifiers, or infrastructure details.
- AI can structure documentation, but the final RCA should still be reviewed by a responsible incident owner.

## Final thoughts

The value of this workflow is not only automation. The main gain is operational consistency: triage starts communication, communication becomes faster and safer, the public status stays aligned with the internal incident stage, incident closure completes the external communication cycle, and the RCA becomes part of a repeatable learning process.

By integrating Jira, Statuspage, Rovo, and Confluence, the team stops treating communication and documentation as separate tasks and starts handling critical incidents with more maturity, predictability, and governance.
