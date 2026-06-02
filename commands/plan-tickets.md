---
description: Plan Jira tickets from tasks.md — writes flat jira-map.md with TBD keys (no Jira creates). Shorthand for taskstotickets plan.
tools:
  - 'Atlassian/getAccessibleAtlassianResources'
  - 'Atlassian/getVisibleJiraProjects'
  - 'Atlassian/getJiraIssue'
  - 'Atlassian/getJiraProjectIssueTypesMetadata'
  - 'Atlassian/createJiraIssue'
  - 'Atlassian/updateJiraIssue'
  - 'Atlassian/searchJiraIssuesUsingJql'
  - 'Atlassian/createJiraIssueLink'
scripts:
  sh: ../scripts/bash/check-prerequisites.sh
  ps: ../scripts/powershell/check-prerequisites.ps1
---

## User Input

```text
plan $ARGUMENTS
```

Execute the full workflow in `commands/taskstotickets.md` with effective invocation
`taskstotickets plan $ARGUMENTS` (`MODE` = **plan**). Do not create Jira issues.
