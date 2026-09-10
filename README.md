# Jira Automation: Create a GitHub Issue and Optionally Assign Copilot

This Jira Automation rule creates a GitHub issue when a Jira work item is assigned.

Each Jira component is treated as the name of a repository in the `OttoPaymentHub` GitHub organization. Before creating the issue, the Jira description is converted from Jira wiki markup into GitHub-compatible Markdown.

The GitHub issue can also be assigned to the GitHub Copilot coding agent through an optional step.

## Automation flow

```text
Work item assigned
├── Check the work item type
├── Check that at least one component is present
├── Check that the work item is assigned
├── Check that the "issue-created" label is not present
└── For each component
    ├── Create the "formattedDescription" variable
    ├── Create the GitHub issue
    ├── Save the GitHub issue number and URL
    ├── Optional: assign the GitHub Copilot coding agent
    ├── Add a comment to the Jira work item
    └── Add the "issue-created" label
```

## Requirements

### General requirements

- Jira component names must exactly match the corresponding GitHub repository names.
- The target repositories must belong to the `OttoPaymentHub` GitHub organization.
- The GitHub token must have access to the target repositories.
- The token must have permission to create and manage issues.
- The token must be stored as a secure value in Jira Automation.
- The `bug` label must exist in every target repository.
- The default branch used by the automation is `main`.

### Optional GitHub Copilot requirements

These requirements only apply when the optional Copilot assignment step is enabled:

- GitHub Copilot coding agent must be enabled for the organization.
- GitHub Copilot coding agent must be enabled for the target repositories.
- The account associated with the token must have access to the Copilot coding agent.
- Copilot must be available as an assignee in the target repository.
- The token must have permission to assign users or agents to issues.

> API permissions alone do not grant access to GitHub Copilot.

---

## 1. Create the trigger

Create a Jira Automation rule with the following trigger:

```text
Trigger → Work item assigned
```

Depending on the Jira version, this trigger may appear as:

```text
Trigger → Issue assigned
```

---

## 2. Add the conditions

Add the following conditions immediately after the trigger.

### 2.1. Check the work item type

Add:

```text
Condition → Work item fields condition
```

Depending on the Jira version, this action may appear as:

```text
Condition → Issue fields condition
```

Configure it as follows:

- **Field:** Issue type
- **Condition:** Is one of
- **Value:** Select the supported work item types

For example:

- Task
- Story
- Bug
- Sub-task

Select only the work item types that should create GitHub issues.

### 2.2. Check that at least one component is present

Add:

```text
Condition → Advanced compare condition
```

Configure:

- **First value:** `{{issue.components}}`
- **Condition:** Does not equal
- **Second value:** Leave empty

This prevents the automation from continuing when no repository can be determined from the Jira components.

### 2.3. Check that the work item is assigned

Add:

```text
Condition → Work item fields condition
```

Configure:

- **Field:** Assignee
- **Condition:** Is not empty

### 2.4. Prevent duplicate executions

Add another advanced comparison condition:

```text
Condition → Advanced compare condition
```

Configure:

- **First value:** `{{issue.labels}}`
- **Condition:** Does not contain
- **Second value:** `issue-created`

The `issue-created` label is added at the end of the automation to prevent the same Jira work item from being processed again.

---

## 3. Create a loop for each Jira component

Immediately after the duplicate-prevention condition, add:

```text
Branch rule / → Advanced branching
```

Configure the branch as follows:

- **Smart value:** `{{issue.components.name}}`
- **Variable name:** `repository`

All the remaining actions must be created inside this branch.

Each branch iteration receives one component name through the `repository` variable. That value is used directly as the GitHub repository name.

For example:

```text
Jira component: payment-service
Branch variable: {{repository}}
GitHub repository: OttoPaymentHub/payment-service
```

> Because `repository` already contains the component name, use `{{repository}}` rather than `{{repository.name}}`.

---

## 4. Create the formatted description variable

As the first action inside the branch, add:

```text
Action → Create variable
```

Configure:

- **Variable name:** `formattedDescription`
- **Smart value:**

````text
{{issue.description.replaceAll("(?m)^h1\\.\\s+(.*)$", "# $1").replaceAll("(?m)^h2\\.\\s+(.*)$", "## $1").replaceAll("(?m)^h3\\.\\s+(.*)$", "### $1").replaceAll("(?m)^h4\\.\\s+(.*)$", "#### $1").replaceAll("(?m)^\\*\\*\\s", "  - ").replaceAll("(?m)^#\\*\\s", "    - ").replaceAll("(?m)^#\\s", "1. ").replaceAll("\\*(.*?)\\*", "**$1**").replaceAll("\\x7Bcode:([^\\x7D]+)\\x7D", "
```$1
").replaceAll("\\x7Bcode\\x7D", "
```
").replaceAll("\\x7Bnoformat\\x7D", "
```
").replaceAll("\\[([^|]+)\\|([^|\\]]+)(?:\\|[^\\]]+)?\\]", "[$1]($2)").replaceAll("!([^|!]+)(?:\\|[^!]+)?!", "`[Image: $1]`").replaceAll("\\x7B\\x7B([^\\x7D]+)\\x7D\\x7D", "`$1`")}}
````

This smart value converts common Jira wiki markup into GitHub-compatible Markdown.

It handles:

- Headings from `h1.` through `h4.`
- Unordered lists
- Nested unordered lists
- Numbered lists
- Bold text
- Code blocks
- No-format blocks
- Jira links
- Jira image references
- Inline code

The resulting value is available in subsequent branch actions as:

```text
{{formattedDescription}}
```

---

## 5. Create the GitHub issue

After creating `formattedDescription`, add:

```text
Action → Send web request
```

### Request configuration

- **Method:** `POST`
- **URL:** `https://api.github.com/repos/OttoPaymentHub/{{repository}}/issues`
- **Web request body:** Custom data

### Headers

Configure the following headers:

```text
Accept: application/vnd.github+json
Authorization: Bearer YOUR_GITHUB_TOKEN
X-GitHub-Api-Version: 2022-11-28
```

Store the `Authorization` value as a secure or hidden value in Jira Automation.

> Do not write the GitHub token directly, add it as a Secret.

### Custom data

```json
{
  "title": "[{{issue.key}}] {{issue.summary.jsonEncode}}",
  "body": "### Ticket description:
{{formattedDescription.jsonEncode}}

### Agent Instructions
Please analyze the described problem and generate a draft Pull Request with the solution.",
  "agent_assignment": {
    "target_repo": "OttoPaymentHub/{{repository}}",
    "base_branch": "main",
    "custom_instructions": "Please analyze the described problem and generate a draft Pull Request with the solution.",
    "custom_agent": "",
    "model": ""
  },
  "labels": [
    "bug"
  ]
}
```

The value of `repository` is already the repository name returned by the branch. Therefore, the `target_repo` value must be:

```text
OttoPaymentHub/{{repository}}
```

Do not use:

```text
OttoPaymentHub/{{repository.name}}
```

### Execution options

Configure:

- **Wait for response:** Yes
- **Continue on error:** No

Waiting for the response is required because subsequent actions use the issue number and URL returned by GitHub.

> The `agent_assignment` object contains configuration for the coding agent. Explicitly assigning Copilot as the issue assignee is handled separately by the optional step described below.
>
> If GitHub returns a validation error for `agent_assignment`, verify that this property is supported by the GitHub API version and features enabled.

---

## 6. Save the GitHub issue response

Immediately after the GitHub issue creation request, save the issue number and URL.

This is particularly important when the optional Copilot assignment is enabled. A second web request replaces the value of `{{webhookResponse}}`.

### 6.1. Save the GitHub issue number

Add:

```text
Action → Create variable
```

Configure:

- **Variable name:** `githubIssueNumber`
- **Smart value:** `{{webhookResponse.body.number}}`

### 6.2. Save the GitHub issue URL

Add another:

```text
Action → Create variable
```

Configure:

- **Variable name:** `githubIssueUrl`
- **Smart value:** `{{webhookResponse.body.html_url}}`

The following values are now available to the remaining branch actions:

```text
{{githubIssueNumber}}
{{githubIssueUrl}}
```

---

## 7. Optional: Assign the issue to GitHub Copilot

> This entire step is optional.
>
> Skip it if GitHub Copilot should not be assigned directly to the newly created issue.

Add:

```text
Action → Send web request
```

### Request configuration

- **Method:** `POST`
- **URL:** `https://api.github.com/repos/OttoPaymentHub/{{repository}}/issues/{{githubIssueNumber}}/assignees`
- **Web request body:** Custom data

### Headers

```text
Accept: application/vnd.github+json
Authorization: Bearer YOUR_GITHUB_TOKEN
X-GitHub-Api-Version: 2022-11-28
```

Configure the `Authorization` value as secure or hidden.

### Custom data

```json
{
  "assignees": [
    "copilot-swe-agent"
  ]
}
```

### Execution options

Because Copilot assignment is optional, the recommended settings are:

- **Wait for response:** Yes
- **Continue on error:** Yes

This allows the automation to continue and update the Jira work item even if Copilot cannot be assigned.

If assigning Copilot must be mandatory, use:

- **Wait for response:** Yes
- **Continue on error:** No

### Troubleshooting the Copilot assignment

If the assignment fails, verify that:

1. GitHub Copilot coding agent is enabled for the organization.
2. GitHub Copilot coding agent is enabled for the target repository.
3. The account associated with the token has access to Copilot.
4. Copilot is available as an assignee in the repository.
5. The token has permission to manage issue assignees.
6. `copilot-swe-agent` is the correct assignee login for the GitHub environment.

---

## 8. Add a comment to the Jira work item

After the GitHub issue has been created, add:

```text
Action → Add comment to work item
```

### Comment when Copilot assignment is enabled

```text
An issue was created for this ticket in the {{repository}} repository.

{{githubIssueUrl}}

Copilot was assigned automatically.
```

### Comment when Copilot assignment is disabled

```text
An issue was created for this ticket in the {{repository}} repository.

{{githubIssueUrl}}
```

The saved `githubIssueUrl` variable is used instead of `{{webhookResponse.body.html_url}}` because the optional Copilot assignment sends another web request and replaces the original `{{webhookResponse}}`.

If the optional Copilot assignment step is not included, you can use the original response directly:

```text
An issue was created for this ticket in the {{repository}} repository.

{{webhookResponse.body.html_url}}
```

> If the Copilot request is configured with **Continue on error: Yes**, do not state that Copilot was assigned unless its response has been validated. Otherwise, the comment could report a successful assignment even when the request failed.

---

## 9. Add the completion label

As the final action inside the component branch, add:

```text
Action → Edit work item fields
```

Open the **Additional fields** section and enter:

```json
{
  "update": {
    "labels": [
      {
        "add": "issue-created"
      }
    ]
  }
}
```

The `issue-created` label indicates that the Jira work item has already been processed and prevents the automation from running again for the same work item.

---

## Final branch structure

All actions shown below must be placed inside the `For each: Smart value` branch, except for the trigger and initial conditions:

```text
Work item assigned
├── Check the work item type
├── Check that components are present
├── Check that the work item is assigned
├── Check that "issue-created" is not present
└── For each smart value
    │   Smart value: {{issue.components.name}}
    │   Variable: repository
    │
    ├── Create variable
    │   Variable: formattedDescription
    │
    ├── Send web request
    │   POST /repos/OttoPaymentHub/{{repository}}/issues
    │
    ├── Create variable
    │   Variable: githubIssueNumber
    │
    ├── Create variable
    │   Variable: githubIssueUrl
    │
    ├── Optional: assign Copilot
    │   POST /repos/OttoPaymentHub/{{repository}}/issues/{{githubIssueNumber}}/assignees
    │
    ├── Add comment to work item
    │
    └── Edit work item fields
        Add label: issue-created
```

---

## Important behavior with multiple components

The branch runs once for every component in:

```text
{{issue.components.name}}
```

For example, if a Jira work item contains these components:

```text
payment-api
payment-ui
```

The automation attempts to create issues in:

```text
OttoPaymentHub/payment-api
OttoPaymentHub/payment-ui
```

A separate Jira comment is added for each GitHub issue created.

### Completion-label consideration

The `issue-created` label is added inside each branch iteration. The initial duplicate-prevention condition is evaluated before the branch starts, so it does not stop the remaining iterations of the same execution.

However, if one repository succeeds and a later repository fails, the Jira work item may still receive the `issue-created` label. A later execution would then be blocked, even though not all repositories were processed successfully.

If processing every component successfully is mandatory, consider moving the final label action outside the branch. Before doing so, verify how your Jira Automation version handles failures inside advanced branches.

---

## Security recommendations

- Store the GitHub token as a secure value in Jira Automation.
- Grant the token only the permissions required to create and assign issues.
- Use a dedicated automation account when possible.

---

## Troubleshooting

### GitHub returns `401 Unauthorized`

Check that:

- The token is valid and has not expired.
- The `Authorization` header uses the correct format.
- The secret is configured correctly in Jira Automation.

### GitHub returns `403 Forbidden`

Check that:

- The token has permission to create and manage issues.
- The account associated with the token can access the repository.
- The GitHub organization allows the selected token type.
- The repository and organization policies permit the requested operation.

### GitHub returns `404 Not Found`

Check that:

- `OttoPaymentHub` is the correct organization name.
- The Jira component exactly matches the GitHub repository name.
- The repository exists.
- The token has access to the repository, especially if it is private.

### GitHub returns `422 Unprocessable Entity`

Check that:

- The `bug` label exists in the target repository.
- The request body contains valid JSON.
- The issue title and body contain valid encoded values.
- The `agent_assignment` property is supported by the API and enabled features.
- The requested assignee is valid for the repository.

### The GitHub description has incorrect formatting

Check the Jira Automation audit log and inspect the value generated for:

```text
{{formattedDescription}}
```

Confirm that the Jira description uses wiki markup compatible with the configured replacements.

The formatting expression handles common Jira wiki syntax but may not convert every Jira element, especially:

- Complex tables
- Panels
- Mentions
- Attachments
- Nested macros
- Rich-text elements stored in Atlassian Document Format

### The Jira comment contains the wrong URL

If the optional Copilot request runs before the comment, do not use:

```text
{{webhookResponse.body.html_url}}
```

The Copilot request replaces the original webhook response. Use the saved variable instead:

```text
{{githubIssueUrl}}
```

### Copilot assignment fails

Check that:

- GitHub Copilot coding agent is enabled for the organization.
- It is enabled for the target repository.
- The token owner has access to the coding agent.
- Copilot is available as an issue assignee.
- `copilot-swe-agent` is the correct login.
- The token can modify issue assignees.

To keep this step optional, configure:

```text
Continue on error: Yes
```

### Duplicate GitHub issues are created

Check that:

- The `issue-created` condition is placed before the component branch.
- The label is added successfully at the end of the automation.
- Multiple executions are not starting simultaneously.
- No other automation rule creates the same GitHub issue.
- The Jira Automation audit log does not show overlapping executions.