---
title: "Advanced Branching in Jira Automation: Why, When, and How to Use It"
date: 2026-02-20T00:00:00-03:00
draft: false
translationKey: "advanced-branching-jira-automation"
tags: ["jira-automation", "advanced-branching", "smart-values", "watchers", "jsm"]
---

If you work with Jira Automation often, you probably already use loops in different ways.

Most of the time, that means iterating through issues: subtasks, linked issues, parents, children, or even results returned by JQL. In other cases, Assets objects can also help solve bulk actions.

But sooner or later, you run into a different kind of problem: you need to iterate over something that is not an issue and not an Assets object.

A common example is when you need to work with watchers, request participants, fixVersions, or organizations. In those cases, the normal branching options are not enough.

That is where **Advanced Branching** becomes useful.

It allows Jira Automation to iterate over smart values that return lists, so you can apply actions to each item in that list.

In this article, I will show what problem it solves, when it makes more sense than normal branching, and how to use it in practice.

---

## The problem

In Jira Automation, the most common options for bulk actions are:

- branching over related issues
- iterating over JQL results
- working with Assets objects through AQL

Those options solve a lot of real cases.

However, there are situations where the data you want to iterate over is not an issue collection. Instead, it comes from a smart value that returns a list.

For example:

- watchers of an issue
- fixVersions
- organizations in Jira Service Management
- request participants

In those cases, **Advanced Branching** is the right tool.

It lets you define a smart value as the source list, assign a variable name to each item, and then use that variable inside the branch actions.

---

## Normal branching vs Advanced Branching

At first glance, both features may look similar, but they are meant for different scenarios.

### Use normal branching when Jira already gives you a defined issue context

Typical examples:

- For each subtask
- For linked issues
- For the parent issue
- For issues returned by JQL

This works well when Jira is already dealing with issues as the main entities.

### Use Advanced Branching when the source is a smart value list

Typical examples:

- `{{issue.watchers}}`
- `{{issue.fixVersions}}`
- `{{issue.organizations}}`

This is the main difference: **Advanced Branching is not about issue relationships. It is about iterating over list-based smart values.**

---

## How to use Advanced Branching in practice

You can find **Advanced Branching** inside Jira Automation when building a rule.

After choosing your trigger, add an **Advanced Branching** component and place your actions inside it.

In this example, the goal is to iterate over the watchers of an issue and send an email to each one.

### Rule structure

```text
Branch: {{issue.watchers}} as watcher
Action: Send email
To: {{watcher.emailAddress}}
```

This is a simple example, but it solves a problem that normal branching cannot solve directly.

### Important note

In Jira Cloud, `emailAddress` may be hidden depending on privacy settings and user visibility rules.

So even though the logic is valid, the result may depend on how your environment handles personal data.

---

## More examples

The same pattern can be used in other lists returned by smart values.

### Iterating over fixVersions

```text
Branch: {{issue.fixVersions}} as version
```

Then inside the branch, you can use values such as:

```text
{{version.name}}
```

### Iterating over organizations

```text
Branch: {{issue.organizations}} as org
```

Inside the branch, you can use the current item variable in the actions that follow.

This is what makes Advanced Branching so useful: once you understand the pattern, you can reuse it in many different scenarios.

---

## Best practices and common pitfalls

Like many Jira Automation features, Advanced Branching is simple once you understand it, but there are still some things worth paying attention to.

### 1. Use clear variable names

Avoid very generic variable names.

Better:

```text
{{issue.watchers}} as watcher
{{issue.fixVersions}} as version
```

Worse:

```text
{{issue.watchers}} as item
```

A clear variable name makes the rule easier to read and maintain.

### 2. Remember the variable scope

The variable created in Advanced Branching only exists inside that branch.

That means you cannot define `watcher` inside the branch and expect to use it later outside of it.

### 3. Handle empty lists

If the list is empty, the branch will simply not iterate.

That is usually fine, but in some cases you may want to check whether the list exists or has values before continuing.

### 4. Watch for duplicates

Depending on the list source, repeated values may appear.

If that matters for your use case, think about how to prevent duplicate actions.

### 5. Check permissions and privacy

This is especially important when dealing with users and email addresses.

The smart value may exist, but the data may still not be available in practice because of privacy controls.

---

## Why this matters

Advanced Branching is one of those Jira Automation features that is easy to ignore at first, but very useful once you understand what it solves.

It helps fill a gap between issue-based branching and list-based smart values.

That makes it a practical solution for scenarios where the data you need is already available in Jira, but not in a form that normal branching can use directly.

---

## Final thoughts

If you already work with Jira Automation, Advanced Branching is worth learning.

It is simple, practical, and solves a type of problem that appears quite often once you start building more advanced rules.

The key idea is this:

- use normal branching for issue relationships
- use Advanced Branching for smart value lists

Once that difference becomes clear, the feature becomes much easier to apply.

#jira #jiraautomation #atlassian #jsm #jiraservicemanagement #automation #smartvalues #advancedbranching
