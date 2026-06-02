---
description: Apply a Jira ticket plan — creates Jira issues for TBD rows in jira-map.md only. Shorthand for taskstotickets apply.
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
apply $ARGUMENTS
```

Execute the full workflow in `commands/taskstotickets.md` with effective invocation
`taskstotickets apply $ARGUMENTS` (`MODE` = **apply**). Requires a prior **plan**.
