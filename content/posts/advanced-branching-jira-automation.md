---
title: "Advanced Branching in Jira Automation: Why, When, and How to Use It"
date: 2026-02-20T00:00:00-03:00
draft: false
tags: ["jira-automation", "advanced-branching", "smart-values", "watchers", "jsm"]
---

If you’re a Jira Administrator, you constantly need to create automations to iterate over lists, arrays, and objects to apply actions in bulk and save time. Most likely, you already use iterations on Issues, looping through subtasks, children, parents, and linked Issues. You can also iterate using Assets objects, and that solved most of the problems. On the other hand, you probably already need an iteration on other properties that were not issues or objects. An example of this is when you need to iterate over watchers or request participants in issues, as you must send an email to everyone.

For this problem, Jira Automation has a solution: Advanced Branching. This feature can use smart values with lists to iterate and apply actions in bulk. Today, we’ll understand why to use, how to use, and examples of problems resolved using this feature.

---

## The problem

In Jira Automation, the intuitive flow, if you want to apply actions in bulk, is AQL (Assets Objects) or a Branch rule / related work items. But, in some cases, you need to iterate over smart values with lists, for example, the issue Watchers. To solve this problem, you can use a feature of Advanced Branching. You define a variable name for the smart value and use it in actions afterward.

---

## Branch normal vs Advanced Branching: when to use each one

Branching normal is used in cases of Jira sending you cases defined, examples:

- For each subtask
- For each issue linked.
- For the parent
- For the issues returned by JQL

Advanced branching is used in cases when you use smart values, and it returns a list, for example:

- Watchers of the issue
- fixVersions
- Organizations of the issue

---

## How to use Advanced Branching in practice (Step by step)

The Advanced Branching feature can be found in Automation for Jira. After choosing a trigger, add Advanced branching and then add your actions inside it. In this case, we’ll use Advanced Branching to iterate a list of watchers and send an email to each one.

The rule will look like this:

- Branch: `{{issue.watchers}}` as `watcher`
- Action: Send email → To: `{{watcher.emailAddress}}`

Note: In Jira Cloud, `emailAddress` may be hidden due to privacy settings.

It’s simple, but solves a big problem, because other types of branching can’t iterate these lists.

You can use in other properties of your Jira Instance, examples:

- `{{issue.fixVersions}}` as `version`
- `{{issue.organizations}}` as `org`

---

## Best Practices and Common Pitfalls

You can follow some best practices to avoid common pitfalls. We will illustrate some of them:

- Variable naming: avoid generic names and collisions with other smart values
- Scope: variable only exists inside the branch
- Empty lists: how to treat with conditions (if, size, isEmpty)
- Deduplication: when the list can repeat users
- Permissions/Privacy: (ex, email may not be exposed)

---

#jira #jiraautomation #atlassian #jsm #jiraservicemanagement #automation #smartvalues #advancedbranching