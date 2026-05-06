# DMTools AI Teammate — Jira Automation Setup

## Prerequisites

- A GitHub account with a repo where you'll set this up (fork or copy [IstiN/dmtools-ai-teammate](https://github.com/IstiN/dmtools-ai-teammate))
- A Jira Cloud instance (Atlassian)
- An AI provider account (Cursor, Codemie, or GitHub Copilot)
- A Google Gemini API key (used as the default LLM)

---

## Step 1 — Fork the Repository

Go to https://github.com/IstiN/dmtools-ai-teammate and fork it into your GitHub org/account.

---

## Step 2 — Create a Dedicated "AI Teammate" Jira Account

1. In your Jira instance, create a new user: e.g. `ai.teammate@yourcompany.com`
2. Note its **Jira Account ID** (visible in the user profile URL or via Jira API)
3. Generate an **Atlassian API Token** at https://id.atlassian.com/manage-profile/security/api-tokens for this account

---

## Step 3 — Add GitHub Repository Secrets

Go to your forked repo → **Settings → Secrets and variables → Actions → Secrets**:

| Secret | Value |
|---|---|
| `JIRA_EMAIL` | The AI Teammate Jira account email |
| `JIRA_API_TOKEN` | Atlassian API token for that account |
| `GEMINI_API_KEY` | Google Gemini API key |
| `CURSOR_API_KEY` | Cursor API key *(or use Codemie/Copilot instead)* |
| `PAT_TOKEN` | GitHub Personal Access Token with `repo` + `actions:write` scopes |

---

## Step 4 — Add GitHub Repository Variables

Go to **Settings → Secrets and variables → Actions → Variables**:

| Variable | Example Value |
|---|---|
| `JIRA_BASE_PATH` | `https://yourcompany.atlassian.net` |
| `JIRA_AUTH_TYPE` | `basic` |
| `CONFLUENCE_BASE_PATH` | `https://yourcompany.atlassian.net/wiki` |
| `CONFLUENCE_DEFAULT_SPACE` | `YOUR_SPACE_KEY` |
| `CONFLUENCE_GRAPHQL_PATH` | `https://yourcompany.atlassian.net/wiki/graphql` |
| `AI_AGENT_PROVIDER` | `cursor` (or `codemie` / `copilot`) |

---

## Step 5 — Customize the Agent Config Files

Edit `agents/story_description.json` and `agents/story_questions.json`:

- Replace `instructions` URLs with your own Confluence pages (or inline text instructions for the AI)
- The `inputJql` and `initiator` fields are overridden at runtime by the Jira webhook — leave them as defaults

---

## Step 6 — Set Up Jira Automation Rule

In Jira: **Project Settings → Automation → Create rule**

**Trigger:** Issue assigned → Assignee is "AI Teammate"

**Condition:** Issue does NOT have label `ai_questions_asked` *(prevents infinite loops)*

**Action:** Send web request with these settings:

- **Method:** `POST`
- **URL:**
  ```
  https://api.github.com/repos/YOUR_ORG/YOUR_REPO/actions/workflows/ai-teammate.yml/dispatches
  ```
- **Headers:**
  ```
  Accept: application/vnd.github.v3+json
  Authorization: token YOUR_PAT_TOKEN
  ```
- **Body** (for story description rewriting):
  ```json
  {
    "ref": "main",
    "inputs": {
      "config_file": "agents/story_description.json",
      "concurrency_key": "{{issue.key}}",
      "encoded_config": "{{#urlEncode}}{
    "params": {
      "inputJql": "key = {{issue.key}}",
      "initiator": "{{initiator.name}}"
    }
  }{{/urlEncode}}"
    }
  }
  ```

> Optionally create a **second rule** using `agents/story_questions.json` in the body to have the AI also generate clarifying question subtasks.

---

## Step 7 — Test It

1. In Jira, take any story ticket and **assign it to the AI Teammate account**
2. Go to your GitHub repo → **Actions** tab — you should see the `ai-teammate` workflow trigger
3. The AI will process the ticket, update the description (or create question subtasks), then reassign it back to you and move it to "In Review"

---

## How it Works

```
Jira ticket assigned to AI Teammate
        ↓
Jira Automation fires webhook → GitHub Actions workflow_dispatch
        ↓
DMTools CLI fetches the ticket via JQL
        ↓
AI agent (Cursor/Codemie/Copilot) processes it per instructions
        ↓
Post-action script: updates Jira description OR creates question subtasks
        ↓
Ticket reassigned to original author + moved to "In Review"
```
