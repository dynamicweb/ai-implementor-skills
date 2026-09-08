# Plugins

Each skill in this repo is packaged as its own Claude Code plugin, so you can install one without
getting the others. This directory holds those packages; `../.claude-plugin/marketplace.json` is the
index that Claude Code reads.

## Available plugins

| Plugin | Skill | Description |
|---|---|---|
| `dw-setup-cli` | `dw-setup-cli` | Operating a Dynamicweb 10 solution with the `dw` CLI — installing custom `.dll`/`.nupkg` add-ins, uploading and updating Files-archive content, exporting files, triggering a recycle, and verifying the result. |
| `dw-extend` | `dw-extend` | Building Dynamicweb and Swift extensions — NotificationSubscribers, Providers, ScheduledTasks, UpdateProviders, database access, Razor overrides, CSS customization, custom ItemTypes — plus debugging on a live solution. |
| `dw-admin-ui` | `dw-admin-ui` | Extending the Dynamicweb 10 administration interface — list/edit/overview screens, areas and area trees, screen injectors, action-menu items — plus deploy and verification. |
| `azure-devops` | `azure-devops` | Azure DevOps REST API — work items, repos, PRs, pipelines, wiki, test plans, security, variable groups, environments, policies. 13 domains, 99 tools. |

See the [root README](../README.md) for installation.

## Layout

A plugin is a directory with a manifest and a `skills/` folder:

```
plugins/<name>/
  .claude-plugin/plugin.json     name, description, version, author, keywords
  skills/<name>/SKILL.md         the skill itself
  skills/<name>/README.md        setup and troubleshooting
  skills/<name>/scripts/         executables the skill calls (azure-devops only)
```

The skill directory name is what Claude Code shows and what you invoke; keeping it equal to the
plugin name avoids confusion.

## Adding a new plugin

1. `mkdir -p plugins/<name>/.claude-plugin plugins/<name>/skills/<name>`
2. Write `plugins/<name>/.claude-plugin/plugin.json` — copy an existing one and change the fields.
3. Write `plugins/<name>/skills/<name>/SKILL.md` with frontmatter (`name`, `description`) and the full
   reference. The `description` is what Claude Code uses to decide whether the skill is relevant, so
   write it as "Use when …".
4. Add a `README.md` next to it with setup and troubleshooting.
5. Add an entry to `../.claude-plugin/marketplace.json` and a row to the table above.
6. Validate both manifests, then check the skill is actually discovered:

   ```bash
   claude plugin validate . && claude plugin validate plugins/<name>
   ```

   Validation passing only means the JSON is well-formed. `claude plugin details <name>` after
   installing is what proves the skill was found — look for `Skills (1)` in the inventory.

## Authentication

`azure-devops` stores a PAT in the system keyring; see its own
[README](azure-devops/skills/azure-devops/README.md). The Dynamicweb plugins need a solution host and
API key — see the `dw-setup-cli` skill.
