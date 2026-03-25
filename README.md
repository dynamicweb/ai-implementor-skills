# 🤖 AI Implementer Skill

A plug-and-play agent configuration for **Claude Code** (and other AI assistants) that gives your AI a complete, Azure DevOps-integrated development workflow — from reading a work item to pushing a PR, without leaving the chat.

---

## ✨ What it does

Drop this into any repo and your AI agent gains:

| Capability | What it means |
|---|---|
| 🎫 **Work item lifecycle** | Read, create, update, link, comment on Azure DevOps work items |
| 🌿 **Branch management** | Create branches from work items with automatic naming (`initials/id-title`) |
| 🔀 **Pull requests** | Create PRs with auto-generated QA checklists, manage reviewers, complete/abandon |
| 🏗️ **Pipeline control** | Trigger, monitor, cancel builds; inspect logs and artifacts |
| 🔍 **Code & wiki search** | Search across code, wiki pages, and work items |
| 📋 **Test plans** | Manage test plans, suites, cases, and results |
| 🔐 **Security alerts** | List and inspect Advanced Security alerts |
| ⚙️ **Variable groups** | Manage pipeline variable groups and secrets |
| 🌍 **Environments & approvals** | Approve/reject deployment approvals |
| 📎 **Attachments** | Upload and attach files to work items |
| 🏛️ **Branch policies** | Create and manage minimum-reviewer and build-validation policies |
| 🧩 **Dynamicweb extension patterns** | Scaffold NotificationSubscribers, Providers, ScheduledTasks, Razor overrides, ViewModel extensions |

The agent handles the **full dev loop autonomously** and pauses only before actions that affect others (push, PR creation).

---

## 📦 What's included

```
.agents/
  ONBOARDING.md                  ← Developer setup guide
  prompts/
    code-review.md               ← Code review prompt
    commit-message.md            ← Commit message generation
    pr-description.md            ← PR description generation
  skills/
    azure-devops/                ← Azure DevOps Python CLI skill
      SKILL.md                   ← Full command reference (99 tools)
      README.md                  ← Setup and troubleshooting
      scripts/                   ← Python scripts (one per domain)
      requirements.txt

.claude/
  settings.json                  ← Claude Code permissions & plugin config
  skills/
    dw-extend.md                 ← Dynamicweb extension scaffolding skill
```

---

## 🚀 Installing into your repo

### 1️⃣ Copy the agent files

```bash
# From within this repo, copy into your target repo
cp -r .agents/ /path/to/your-repo/
cp -r .claude/ /path/to/your-repo/
```

### 2️⃣ Install Python dependency

```bash
pip install keyring>=24.0.0
```

Or from the skill directory:

```bash
pip install -r .agents/skills/azure-devops/requirements.txt
```

### 3️⃣ Authenticate with Azure DevOps

**Option A — OAuth (recommended, tokens auto-refresh):**
```bash
cd .agents/skills/azure-devops
python scripts/auth.py login --org YourOrganization
# Follow the URL and enter the device code
```

**Option B — Personal Access Token:**
```bash
python scripts/auth.py login --org YourOrganization --pat YOUR_PAT
```

Create a PAT at `https://dev.azure.com/{org}/_usersSettings/tokens` with these scopes:
- Work Items: Read & Write
- Code: Read & Write
- Build: Read & Execute
- Wiki: Read & Write
- Test Management: Read & Write
- Advanced Security: Read
- Project and Team: Read
- Identity: Read

### 4️⃣ Store your personal details in Claude memory

Open Claude Code in the repo and run:

```
Remember my developer initials as: <your initials>
Remember my Azure DevOps PAT as: <your PAT>
```

Your initials are used for branch naming (`np/27641-fix-login`). The agent will ask if they're missing.

### 5️⃣ Verify it works

```bash
python .agents/skills/azure-devops/scripts/auth.py status
python .agents/skills/azure-devops/scripts/core.py list-projects
```

---

## 🗣️ Using the agent

You don't need to spell out every step. The agent knows the full workflow.

### Starting from a work item

```
I want to work on work item #27641. Read it, create a branch, and set it to Active.
```

```
Work item #27641. Read it, implement the fix, commit, push and create a PR. Work item is a child of #27000.
```

### Starting from a GitHub issue

```
<paste GitHub issue URL>

Read this issue. Create a work item for it as a child of #27000, create a branch, implement the fix, commit, push and create a PR.
```

### Starting from a bug report

```
Here is a bug report from our forum:

<paste bug description>

Create a work item for this as a child of #27000, create a branch, implement the fix, commit, push and create a PR.
```

### Code review

```
Review the changes on this branch against master. Check for violations of project conventions, API breaking changes, and anything QA should know about.
```

### Explain code

```
Explain how <class or feature name> works, and where it fits in the solution architecture.
```

---

## 🔧 Azure DevOps skill — domains

The skill covers **13 domains** and **99 tools** via the Azure DevOps REST API v7.1:

| Domain | Script | Tools |
|--------|--------|-------|
| Core | `core.py` | Projects, teams, identities (3) |
| Work Items | `work_items.py` | CRUD, queries, comments, backlogs (20) |
| Iterations | `work.py` | Sprints, capacity (7) |
| Git/PRs | `repos.py` | Repos, branches, pull requests (18) |
| Pipelines | `pipelines.py` | Builds, runs, artifacts (14) |
| Search | `search.py` | Code, wiki, work item search (3) |
| Wiki | `wiki.py` | Pages management (6) |
| Test Plans | `test_plans.py` | Plans, suites, cases, results (9) |
| Security | `security.py` | Advanced security alerts (2) |
| Variable Groups | `variable_groups.py` | Pipeline variable groups (7) |
| Environments | `environments.py` | Environments & approvals (8) |
| Policies | `policies.py` | Branch policies (8) |
| Attachments | `attachments.py` | Work item attachments (6) |

See [`.agents/skills/azure-devops/SKILL.md`](.agents/skills/azure-devops/SKILL.md) for the full command reference.

---

## 🧩 Dynamicweb extension skill

The `dw-extend` skill guides the agent in building **upgrade-safe** Dynamicweb and Dynamicweb Swift extensions:

- **NotificationSubscribers** — hook into platform events (order placed, user login, page render)
- **Providers** — override core calculations (price, tax, shipping, stock, feed)
- **ScheduledTask AddIns** — run logic on a schedule or via the admin task runner
- **Swift CSS / Razor overrides** — change frontend presentation without touching Swift core
- **ViewModel extensions** — add custom data to Razor template contexts
- **Administration UI extensions** — custom admin screens and buttons
- **UpdateProviders** — custom database schema migrations

Invoke it by asking Claude Code to build or scaffold a Dynamicweb extension.

---

## 🛡️ What the agent will NOT do without your permission

- Merge pull requests
- Force-push to any branch
- Modify pipeline YAML
- Push commits or create PRs (always pauses and confirms first)

---

## 🐛 Troubleshooting

| Error | Fix |
|---|---|
| `Not authenticated` | Run `python scripts/auth.py login` |
| `HTTP 401` | Token expired — re-run `auth.py login` |
| `HTTP 403` | PAT missing a required scope — recreate with correct scopes |
| `HTTP 404` | Wrong project name, repo name, or resource ID |
| Keyring issues on Linux | `pip install secretstorage` |

---

## 📄 License

Apache-2.0
