# Jira Automation: Create a GitHub Issue and Optionally Assign Copilot

This Jira Automation rule creates a GitHub issue whenever a Jira issue is assigned.

Each Jira component is treated as the name of a GitHub repository. Optionally, the newly created GitHub issue can be assigned to the GitHub Copilot coding agent.

## Automation flow

```text
Issue assigned
├── Check the issue type
├── Check that at least one component is present
├── Check that an assignee is present
├── Check that the "issue-created" label is not present
└── For each component
    ├── Create a GitHub issue
    ├── Save the GitHub issue number and URL
    ├── Optional: assign GitHub Copilot
    ├── Comment on the Jira issue
    └── Add the "issue-created" label
```

## Requirements

### General requirements

- Jira component names must exactly match the corresponding GitHub repository names.
- The GitHub token must have access to the target repositories.
- The token must have permission to create and manage issues.
- The token must be stored as a secure value in Jira Automation.
- The `bug` label must exist in every target repository if it is included in the request.

### Optional GitHub Copilot requirements

The following requirements only apply if GitHub issues should be assigned automatically to the GitHub Copilot coding agent:

- GitHub Copilot coding agent must be enabled for the organization.
- GitHub Copilot coding agent must be enabled for the target repositories.
- The account associated with the token must have access to GitHub Copilot coding agent.
- GitHub Copilot must be available as an assignee in the target repository.

> **Important:** API permissions alone do not grant access to GitHub Copilot.

---

## 1. Create the trigger

Create a Jira Automation rule with the following trigger:

```text
Trigger → Issue assigned
```

---

## 2. Add the conditions

### 2.1. Check the issue type

Add:

```text
Condition → Issue fields condition
```

Configure the condition as follows:

- **Field:** Issue type
- **Condition:** Is one of
- **Value:** Select the supported issue types

If you are importing an existing rule, verify that the issue type IDs match the issue types in your Jira instance. IDs may differ between environments.

### 2.2. Check that a component is present

Add:

```text
Condition → Advanced compare condition
```

Configure:

- **First value:** `{{issue.components}}`
- **Condition:** Does not equal
- **Second value:** Leave empty

### 2.3. Check that an assignee is present

Add:

```text
Condition → Issue fields condition
```

Configure:

- **Field:** Assignee
- **Condition:** Is not empty

### 2.4. Prevent duplicate executions

Add:

```text
Condition → Advanced compare condition
```

Configure:

- **First value:** `{{issue.labels}}`
- **Condition:** Does not contain
- **Second value:** `issue-created`

The `issue-created` label will be added after the automation completes successfully.

---

## 3. Create a branch for each repository

Add an advanced branch:

```text
Branch rule / related issues → Advanced branching
```

Configure:

- **Smart value:** `{{issue.components.name}}`
- **Variable name:** `repository`

All subsequent actions must be added inside this branch.

Each Jira component will be treated as a GitHub repository name.

> **Example:** A Jira component named `payment-service` maps to a GitHub repository named `payment-service`.

---

## 4. Create the GitHub issue

Add:

```text
Action → Send web request
```

### Request

- **Method:** `POST`
- **URL:** `https://api.github.com/repos/GITHUB_ORG/{{repository}}/issues`
- **Body type:** Custom data

Replace `GITHUB_ORG` with the name of your GitHub organization.

For example:

```text
https://api.github.com/repos/OttoPaymentHub/{{repository}}/issues
```

### Headers

```text
Accept: application/vnd.github+json
Authorization: Bearer YOUR_GITHUB_TOKEN
X-GitHub-Api-Version: 2022-11-28
```

Configure the `Authorization` header as a secure or hidden value.

> Never write the token directly in this README or include it in an exported Jira Automation file.

### Request body

```json
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
```

### Execution options

- **Wait for response:** Yes
- **Continue on error:** No

> If the `bug` label does not exist or cannot be applied by the token, GitHub may reject the request. Remove the `labels` property if the label is not required.

---

## 5. Save the GitHub response

Immediately after creating the issue, add two `Create variable` actions.

These values must be saved before sending another web request because a subsequent request will replace `{{webhookResponse}}`.

### 5.1. Save the issue number

- **Variable name:** `githubIssueNumber`
- **Value:** `{{webhookResponse.body.number}}`

### 5.2. Save the issue URL

- **Variable name:** `githubIssueUrl`
- **Value:** `{{webhookResponse.body.html_url}}`

---

## 6. Optional: Assign the issue to GitHub Copilot

> **This entire section is optional.**
>
> If you do not want to assign GitHub issues automatically to the Copilot coding agent, skip this step and continue with section 7.

Add:

```text
Action → Send web request
```

### Request

- **Method:** `POST`
- **URL:** `https://api.github.com/repos/GITHUB_ORG/{{repository}}/issues/{{githubIssueNumber}}/assignees`
- **Body type:** Custom data

Replace `GITHUB_ORG` with the name of your GitHub organization.

### Headers

```text
Accept: application/vnd.github+json
Authorization: Bearer YOUR_GITHUB_TOKEN
X-GitHub-Api-Version: 2022-11-28
```

Configure the `Authorization` header as a secure or hidden value.

### Request body

```json
{
  "assignees": [
    "copilot-swe-agent"
  ]
}
```

> Confirm the correct Copilot assignee login for your GitHub environment. Depending on the GitHub configuration and API behavior, the displayed account name may differ.

### Execution options

Recommended configuration if Copilot assignment is optional:

- **Wait for response:** Yes
- **Continue on error:** Yes

Setting **Continue on error** to **Yes** ensures that a Copilot assignment failure does not prevent the Jira comment and completion label from being added.

If Copilot assignment must succeed before the automation can continue, use:

- **Wait for response:** Yes
- **Continue on error:** No

### Troubleshooting the Copilot assignment

If the assignment fails, verify that:

1. GitHub Copilot coding agent is enabled for the organization.
2. GitHub Copilot coding agent is enabled for the repository.
3. The account associated with the token has access to Copilot.
4. Copilot is available as an assignee in the repository.
5. The token has permission to manage repository issues.
6. The assignee login used in the request is correct.

---

## 7. Add a comment to the Jira issue

Add:

```text
Action → Comment on issue
```

### Comment when Copilot assignment is enabled

```text
A GitHub issue was created in the {{repository}} repository:

{{githubIssueUrl}}

GitHub Copilot coding agent was assigned automatically.
```

### Comment when Copilot assignment is disabled

```text
A GitHub issue was created in the {{repository}} repository:

{{githubIssueUrl}}
```

> If Copilot assignment uses **Continue on error: Yes**, avoid stating that Copilot was assigned unless the response was validated successfully.

---

## 8. Add the completion label

Add:

```text
Action → Edit issue
```

Add the following label:

```text
issue-created
```

Alternatively, use the advanced JSON configuration:

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

This label indicates that the Jira issue has already been processed and helps prevent duplicate executions.

---

## Multiple components

If a Jira issue contains multiple components:

- One GitHub issue will be created in each corresponding repository.
- The `repository` variable will contain a different repository name in each branch iteration.
- One Jira comment will be added for each GitHub issue.
- If Copilot assignment is enabled, each GitHub issue will be assigned separately.

Ensure that every Jira component represents a valid and accessible GitHub repository. Otherwise, GitHub will return an error for the corresponding branch.

---

## Security recommendations

- Do not store the GitHub token directly in this file.
- Do not include the token in repositories or exported automation files.
- Store the `Authorization` header as a secure value in Jira Automation.
- Grant the token only the permissions required by this automation.
- Use a dedicated automation account where possible.
- Define an expiration and rotation policy for the token.
- Test token access using a non-production repository first.

---

## Recommended test procedure

Before enabling the rule in production:

1. Use a non-production Jira project.
2. Use a non-production GitHub repository.
3. Confirm that the Jira component name matches the repository name.
4. Assign a test Jira issue.
5. Verify that the GitHub issue is created.
6. Verify that its number and URL are saved correctly.
7. If enabled, verify that Copilot is assigned.
8. Confirm that the expected comment is added to Jira.
9. Confirm that the `issue-created` label is added.
10. Reassign the Jira issue and confirm that no duplicate issues are created.

---

## Troubleshooting

### GitHub returns `401 Unauthorized`

- Confirm that the token is valid.
- Check the format of the `Authorization` header.
- Verify that the token has not expired.

### GitHub returns `403 Forbidden`

- Check the token permissions.
- Confirm that the account has access to the repository.
- Review the organization's security and token policies.

### GitHub returns `404 Not Found`

- Check the GitHub organization name.
- Confirm that the Jira component matches the repository name.
- Verify that the token can access private repositories.

### Copilot assignment fails

- Confirm that Copilot coding agent is enabled.
- Verify that Copilot is available as an assignee.
- Check that the account associated with the token has Copilot access.
- Confirm the correct Copilot assignee login.
- Use **Continue on error: Yes** if Copilot assignment should remain optional.

### Duplicate GitHub issues are created

- Ensure that the `issue-created` condition is evaluated before entering the branch.
- Verify that the completion label is added successfully.
- Check whether multiple rule executions are running concurrently.
- Review the Jira Automation audit log for overlapping executions.
