---
title: "How Rovo Agent Helps Support Analysts Resolve Tickets Faster"
date: 2026-01-23T00:00:00-03:00
draft: false
tags: ["jira-automation", "rovo", "Support Analyst"]
---

Slower ticket resolution is a big problem in lots of companies. First, a ticket is received by the Support Analyst team (N1). After that, tickets go to the Development Analyst team, where they do a triage. In the end, the ticket goes back to N1 Support Analyst because the information is incomplete, there are other tickets with the same bug already resolved or in progress, or the company has a knowledge base and the content can solve this ticket without needing a technical team.

In this post, I'll show how to use Rovo Agent, Atlassian’s Artificial Intelligence, to guide your Support Analyst team (N1) to understand the ticket, classify it, make hypotheses, search solutions in your Knowledge Base, share similar tickets, and say when to escalate.

---

## The pain

- Escalated with incomplete information.
- Duplicated tickets being resolved by the technical team.
- Knowledge base has information that can solve this ticket.
- Previous tickets resolved with the same problem.

---

## The goal

What success looks like:

- Complete triage of your ticket.
- Collect all information necessary for your technical team.
- Search and display content from your knowledge base with the capacity to solve your ticket.
- Display hypotheses about your ticket problem.

---

## What Rovo changes in practice

- **Before:** N1 escalates quickly → Dev asks for missing data → ticket bounces back.
- **After:** N1 gets a structured checklist, links to playbook pages, and similar-ticket clues **before** escalation.

A good agent reduces:
- Back-and-forth comments
- Duplicate investigations
- Time to first meaningful action (TTFA)

---

## How to create your first Agent

1. Go to your Jira Cloud instance and, in the left menu, click on Atlassian apps content. In this section, you can see the Studio section. This part is where Agents are created.
2. After that, you can create your Rovo agent. Atlassian provides a very intuitive form with questions about your agent: name, purpose, objective, etc. In the end, it provides your agent with context from your organization.
3. In the next step, you can go to Global Automation and create a new automation. Its structure is:  
   Trigger -> Rovo -> Action with the message from Rovo.
4. Trigger example: Issue Created in the JSM project.
5. Action: Use Rovo Agent. In this step, you must select your agent and, in the main action, place your prompt.

Your prompt is very important, because it uses information from your context and company that will help solve your tickets.

Example prompt:

```text
Generic Internal Support Assistant Prompt

You are an internal Support assistant for [Company Name].
You NEVER write to the end-customer. You respond only to the ticket analyst with practical triage and resolution guidance.

Sources (mandatory order)

Knowledge Base / Playbook: space “[Support Playbook / Runbook]” (official procedures)

Ticketing System: similar tickets from the same project/service (same topic/error/customer/provider)

Task

When you receive a ticket, do the following:

Read the entire ticket: summary, description, fields (customer/brand, category, impact, affected count, SLA, squad/team), comments, and any referenced attachments/logs/screenshots.

Understand and classify the request: Incident / Question / Request / Bug. If unclear, mark Hypothesis.

Search the Playbook using ticket keywords (product, provider, error code, workflow terms like “withdrawal”, “cashout”, “face verification”, “liveness”, “V001”, etc.). Extract relevant checklists, steps, criteria, and required evidence.

Search similar tickets in the same project and reuse only:

questions that unblocked the case

validations/checks performed

proven resolution paths

Do not invent policies, timelines, root causes, limitations, or commitments.

If data is missing, declare Gap and specify exactly what to request/check.

Output rules

Be brief: max 12–15 lines, bullet points, no long paragraphs.

Always prioritize the Playbook first; only then use similar tickets.

If there is no reliable basis: write “No reliable basis” and recommend escalation.

Mandatory link rule in “References”

In block 5) References, every reference must be a clickable Markdown link.

Playbook: use the URL returned by the search (page URL).

Tickets: use the URL returned by the search (ticket URL).

Forbidden: inventing URLs.

If search does not return a URL, write: “(no link returned by search)” and include the keywords used.

Link format (mandatory)

Playbook: Playbook: <page title>
 — <why it matters in 1 line>

Ticket: <TICKET-ID>
 — <what was useful in 1 line>

Mandatory response format (always exactly these blocks)

Summary (1 sentence)

…

Data/Evidence to collect

…

Immediate checks (step-by-step)

…

…

…

Likely hypotheses (mark as hypothesis)

Hypothesis: …

Hypothesis: …

References

Playbook: …

Ticket: …

Escalate when

Objective criterion: … → Suggested team: …

Objective criterion: … → Suggested team: …
```

6. After this, your automation will use Rovo and receive a smart value: `{{agentResponse}}`.
7. In the last step, you can use the value as a comment in your company. In this case, Rovo helps an internal Support Analyst, so your comment should be internal too.

---

## Automation wiring example

A simple flow:

- **Trigger:** Issue Created (JSM project)
- **Condition (optional):** Only when `Request type` is X / or `Labels` contains Y
- **Action:** Use Rovo Agent (your prompt)
- **Action:** Add internal comment with `{{agentResponse}}`

Tip: Also store the response in a custom field (optional) for reporting.

---

## What a “good” agent response looks like

- 1-sentence summary
- Exactly what data is missing (Gap)
- Step-by-step checks N1 can do now
- 2–4 hypotheses (clearly marked)
- References: playbook + similar tickets (with links)
- Clear escalation criteria + suggested team

---

## Checklist before you ship

- [ ] Does the rule exit safely if triggered twice?
- [ ] Does it avoid repeating create operations?
- [ ] Are failures stored somewhere (property, comment, field)?
- [ ] Is there a stale timeout to avoid stuck locks?
- [ ] Do retries have a cap?

## Optional: add lock + idempotency for safety

If people can trigger the rule manually (or it can run twice), you can add:
- **Lock per issue** (store “in-progress” state somewhere)
- **Idempotency key** (store “already processed” state)

If you implement just Pattern 1 (lock) and Pattern 2 (idempotency), you already eliminate most duplicate execution pain.
