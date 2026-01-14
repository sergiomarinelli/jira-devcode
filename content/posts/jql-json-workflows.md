---
title: "JQL + JSON: building reliable Jira automations"
date: 2026-01-13
draft: false
tags: ["jira", "jsm", "automation", "jql", "api"]
---

This post is a starting point for Jira DevCode: patterns, gotchas, and reliable automation design.

## What you’ll learn
- JQL patterns that scale
- Automation rule architecture
- REST API workflows (safe retries, idempotency)

## Example: JQL baseline
```jql
project = ABC AND statusCategory != Done ORDER BY created DESC
