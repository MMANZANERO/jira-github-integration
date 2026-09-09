# Jira Automation: Create a GitHub Issue and Assign Copilot
This automation creates a GitHub issue when a Jira issue is assigned. It uses each Jira component as a GitHub repository name and then assigns the issue to GitHub Copilot coding agent.

## Requirements
Jira component names must match the GitHub repository names.
The GitHub token must have access to the target repositories and permission to manage issues.
The account associated with the token must have access to GitHub Copilot coding agent.
Copilot coding agent must be enabled for the organization and repositories.
Store the token as a secure value in Jira Automation.
Automation structure
Issue assigned
├── Check issue type
├── Check components
├── Check assignee
├── Check that "issue-created" is not present
└── For each component
    ├── Create GitHub issue
    ├── Save GitHub issue number and URL
    ├── Assign Copilot
    ├── Comment on Jira
    └── Add "issue-created" label
The original rule uses the Issue assigned trigger and iterates over {{issue.components.name}}. [1]

1. Trigger
Create a Jira Automation rule with:

Trigger → Issue assigned
2. Conditions
Add the following conditions:

Allowed issue types
Condition → Issue fields condition

Field: Issue type
Condition: Is one of
Value: Select the supported issue types
The exported rule uses issue-type IDs 10006, 10100, 10101, and 10007. [1]

Component is present
Condition → Advanced compare condition

First value: {{issue.components}}
Condition: Does not equal
Second value: Leave empty
Assignee is present
Condition → Issue fields condition

Field: Assignee
Condition: Is not empty
Prevent duplicates
Condition → Advanced compare condition

First value: {{issue.labels}}
Condition: Does not contain
Second value: issue-created
The issue-created label is used by the exported rule to prevent duplicate executions. [1]

3. Branch over repositories
Add an advanced branch:

Smart value: {{issue.components.name}}
Variable name: repository
All subsequent actions must be added inside this branch. Each component name will be treated as a repository name. [1]

4. Create the GitHub issue
Add:

Action → Send web request
Request
Method: POST
URL: https://api.github.com/repos/OttoPaymentHub/{{repository}}/issues
Body type: Custom data
Replace OttoPaymentHub with your GitHub organization if necessary.

Headers
Accept: application/vnd.github+json
X-GitHub-Api-Version: 2026-03-10
Authorization: Bearer YOUR_GITHUB_TOKEN
Mark Authorization as a secure value.

Body
{
  "title": "[{{issue.key}}] {{issue.summary.jsonEncode}}",
  "body": "### Ticket description

{{issue.description.jsonEncode}}

### Instructions

Please analyze this issue, implement the required changes, run the relevant tests, and create a draft pull request.",
  "labels": [
    "bug"
  ]
}
Enable:

Wait for response: Yes
Continue on error: No
The exported rule creates an issue through this GitHub endpoint and uses the Jira component as the repository name. [1]

5. Save the GitHub response
Immediately after creating the issue, add two Create variable actions.

Issue number
Variable name: githubIssueNumber
Value: {{webhookResponse.body.number}}
Issue URL
Variable name: githubIssueUrl
Value: {{webhookResponse.body.html_url}}
These values must be saved before sending another web request because {{webhookResponse}} will be replaced.

6. Assign Copilot coding agent
Add another:

Action → Send web request
Request
Method: POST
URL: https://api.github.com/repos/OttoPaymentHub/{{repository}}/issues/{{githubIssueNumber}}/assignees
Body type: Custom data
Headers
Accept: application/vnd.github+json
X-GitHub-Api-Version: 2026-03-10
Authorization: Bearer YOUR_GITHUB_TOKEN
Body
{
  "assignees": [
    "copilot-swe-agent"
  ]
}
Enable:

Wait for response: Yes
Continue on error: No
The account associated with the token must have access to GitHub Copilot coding agent. API permissions alone do not provide Copilot access.

7. Comment on the Jira issue
Add:

Action → Comment on issue
Comment:

A GitHub issue was created in the {{repository}} repository:

{{githubIssueUrl}}

GitHub Copilot coding agent was assigned automatically.
8. Add the completion label
Add:

Action → Edit issue
Add the following label:

issue-created
Alternatively, use advanced JSON:

{
  "update": {
    "labels": [
      {
        "add": "issue-created"
      }
    ]
  }
}
The exported automation adds this label after creating the GitHub issue and Jira comment. [1]

Final notes
Test the rule with a non-production repository first.
The bug label must exist in each target repository.
Do not store the GitHub token directly in this README or in the exported automation file.
If Copilot assignment fails, confirm that Copilot coding agent is enabled and available as an assignee in the repository.
