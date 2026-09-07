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
| 🚚 **Deploy to a solution** | Install add-ins, push and pull Files-archive content, trigger a recycle, verify it landed |
| 🖥️ **Admin interface extensions** | Build list/edit/overview screens, add areas and tree nodes, inject into screens you don't own |

The agent handles the **full dev loop autonomously** and pauses only before actions that affect others (push, PR creation).

---

## 📦 What's included

Each skill is packaged as its own **Claude Code plugin**, so you can install one without getting the
others. This repo is also the marketplace that indexes them.

```
.claude-plugin/
  marketplace.json                 ← the plugin index Claude Code reads
plugins/
  README.md                        ← plugin index for humans
  azure-devops/                    ← Azure DevOps REST API (13 domains, 99 tools)
    .claude-plugin/plugin.json
    skills/azure-devops/           ← SKILL.md  README.md  requirements.txt  scripts/
  dw-extend/                       ← Dynamicweb & Swift extension patterns
    .claude-plugin/plugin.json
    skills/dw-extend/              ← SKILL.md  README.md
  dw-cli/                          ← Deploying to a solution with the `dw` CLI
    .claude-plugin/plugin.json
    skills/dw-cli/                 ← SKILL.md  README.md
  dw-admin-ui/                     ← Extending the Dynamicweb admin interface
    .claude-plugin/plugin.json
    skills/dw-admin-ui/            ← SKILL.md  README.md
.claude/
  ONBOARDING.md                    ← Developer setup guide
  settings.json                    ← permissions, and the plugins enabled in this repo
  prompts/                         ← code-review.md  commit-message.md  pr-description.md
```

A `SKILL.md` with YAML frontmatter is what Claude Code discovers; the sibling `README.md` is for humans.
The `plugin.json` is what makes the directory installable.

---

## 🚀 Installing into your repo

### 1️⃣ Install the skills you want

Register this repo as a marketplace once, then install per skill. In Claude Code:

```
/plugin marketplace add dynamicweb/ai-implementor-skills
/plugin install dw-cli@dynamicweb
```

Or from a terminal:

```bash
claude plugin marketplace add dynamicweb/ai-implementor-skills
```

```bash
claude plugin install dw-cli@dynamicweb
```

**Install one, or several** — `dw-cli`, `dw-extend`, `dw-admin-ui`, `azure-devops`. They do not depend on
each other; they cross-reference in prose (`dw-admin-ui` points at `dw-cli` for deployment), and an absent
skill just means that pointer goes unused.

`--scope` decides who gets it: `user` (default, all your projects), `project` (written to the repo's
`.claude/settings.json`, so everyone who opens that repo gets the skill), or `local` (just you, just this
repo).

**Or point Claude Code at the repo in plain language.** It is public, so you can say:

```
Install the dw-cli skill from https://github.com/dynamicweb/ai-implementor-skills
```

Either way, the authentication steps below are yours to run, since they involve credentials.

**Verify the skill was actually found.** Installation reporting success is not the same as discovery:

```bash
claude plugin details dw-cli
```

Look for `Skills (1)  dw-cli` in the component inventory. The same command reports the token cost — each
of these adds roughly 60–120 tokens to every session, and 4–20k when it fires.

<details>
<summary>Copying the files instead</summary>

A skill directory is self-contained, so copying it works too. You lose versioning and
`claude plugin update`, but nothing else:

```bash
cp -r plugins/dw-cli/skills/dw-cli /path/to/your-repo/.claude/skills/
```

Claude Code discovers any `.claude/skills/<name>/SKILL.md` in a repo automatically — no registration, no
restart.
</details>

### 2️⃣ Optional: permissions

A skill works without any settings change — you will just be asked to approve each shell command it runs.
To stop the prompting, add the commands that skill uses to your own
`.claude/settings.json`:

| Skill | Worth allowing |
|---|---|
| `azure-devops` | `Bash(python *)`, `Bash(cd * && python *)` |
| `dw-extend`, `dw-admin-ui` | `Bash(dotnet *)` |
| `dw-cli` | `Bash(dw *)`, `Bash(npm *)` |
| any of them | `Bash(curl *)` for reading the Management API |

```json
{
  "permissions": {
    "allow": ["Bash(dotnet *)", "Bash(dw *)"]
  }
}
```

> **Do not copy this repo's `settings.json` wholesale into an existing repo.** Besides permissions it
> sets `enabledPlugins` for twelve Claude Code plugins — eight from the official marketplace plus the
> four in this repo — which will turn those on for whoever opens it. Merge the `permissions.allow` entries
> you want into your own file instead.
>
> Note this repo's own `settings.json` predates the `dw-cli` skill and does **not** allow `Bash(dw *)`,
> so `dw` commands still prompt here.

### 3️⃣ Install what each skill needs

| Skill | Requires | Install |
|---|---|---|
| `azure-devops` | Python 3.10+ and `keyring` | `pip install -r plugins/azure-devops/skills/azure-devops/requirements.txt` |
| `dw-extend` | .NET SDK 10.0 to build extensions | [dotnet.microsoft.com](https://dotnet.microsoft.com/download) |
| `dw-admin-ui` | .NET SDK 10.0 | as above |
| `dw-cli` | Node.js ≥ 20.12 and the `dw` CLI | `npm install -g @dynamicweb/cli` |

Only `azure-devops` ships scripts; the other three are knowledge skills with nothing to run.

### 4️⃣ Authenticate with Azure DevOps

*Only needed for the `azure-devops` skill.*

**Option A — OAuth (recommended, tokens auto-refresh):**
```bash
cd plugins/azure-devops/skills/azure-devops
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

### 5️⃣ Authenticate against a Dynamicweb solution

*Only needed for `dw-cli` and `dw-admin-ui`.*

Create an API key in the solution's admin under **Settings → System → Developer → Api Keys**, then store
it in a file rather than pasting it into a command line:

```bash
printf '%s' 'YOUR_KEY' > ~/.dw-apikey
```

Every `dw` command then reads it at call time:

```bash
dw files ./templates Templates -i -o --host <solution>.dynamicweb.cloud --apiKey "$(cat ~/.dw-apikey)"
```

`dw login` is not used — it is unavailable on `*.dynamicweb.cloud` solutions. See the `dw-cli` skill.

> ⚠️ **Do not pass `--apiKey` to `dw query` or `dw command`.** Those two leak the key into the request
> URL and print it in their error output. Use `dw files` / `dw install` normally, and call the Management
> API yourself with an `Authorization: Bearer` header for anything else.

### 6️⃣ Store your personal details in Claude memory

Open Claude Code in the repo and run:

```
Remember my developer initials as: <your initials>
```

Your initials are used for branch naming (`np/27641-fix-login`). The agent will ask if they're missing.

### 7️⃣ Verify it works

```bash
claude plugin list
python plugins/azure-devops/skills/azure-devops/scripts/auth.py status
python plugins/azure-devops/skills/azure-devops/scripts/core.py list-projects
npm ls -g @dynamicweb/cli
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

See [`plugins/azure-devops/skills/azure-devops/SKILL.md`](plugins/azure-devops/skills/azure-devops/SKILL.md) for the full command reference.

---

## 🧩 The Dynamicweb skills

### `dw-extend` — extension patterns

Building **upgrade-safe** extensions for Dynamicweb and Dynamicweb Swift:

- **NotificationSubscribers** — hook into platform events
- **Providers** — override core calculations (price, tax, shipping, stock, feed)
- **ScheduledTask AddIns** — run logic on a schedule or via the admin task runner
- **UpdateProviders** — custom database schema migrations, and how to debug them
- **Swift CSS / Razor overrides** — change frontend presentation without touching Swift core
- **ViewModel extensions** — add custom data to Razor template contexts
- **Repositories** — search indexes, queries and facets

Also carries a general **debugging** section: reading the `GeneralLog` table, querying it over the
Management API, and running arbitrary SQL against a solution.

### `dw-admin-ui` — the administration interface

Extending the admin ("backend") from your own assembly:

- **Screens** — list, edit and overview, each backed by a query, a command and a data model
- **Navigation** — your own area in the sidebar, sections and nodes in its tree
- **Extending what you don't own** — screen injectors, nodes under someone else's tree node, entries in
  an existing Actions menu

Every C# pattern in it is compile-verified, and the whole chain has been deployed and confirmed rendering
on a live solution.

### `dw-cli` — getting it onto a solution

Operating a solution with the `dw` CLI: installing `.dll`/`.nupkg` add-ins, uploading and updating
templates and other Files content, exporting the archive, and triggering a recycle. Includes the
verification steps that matter, because **`dw install` reports success whether or not your code loaded**.

Invoke any of them by asking Claude Code for the task — they trigger on intent, not by name.

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
