# Dynamicweb CLI Skill

Operating a live Dynamicweb 10 solution from the command line with `dw` — installing custom add-ins and
moving Files-archive content in and out.

## What this skill covers

- **Add-in install loop** — `dotnet build` → `dw install` → verify the add-in loaded on the solution
- **Uploading and updating files** — `dw files -i`, including the overwrite semantics that decide whether
  an update actually happens
- **Exporting files** — `dw files -e` for directories or single files, with file/directory detection
- **Listing** — `dw files -l` as the verification step for both workflows
- **Preflight** — version differences, Git Bash path conversion, environment confirmation, auth handoff

## MCP or the CLI?

One question decides it: **are you changing a file, or a record?**

| Changing | Use |
|---|---|
| Template, stylesheet, asset, add-in, marker file | **`dw` CLI** — this skill |
| Page, paragraph, item field, product, user, order | **Dynamicweb MCP** |

They barely overlap. MCP's `upload_file` writes only to `/Files/Images` and `/Files/Files/Integration`
— templates are explicitly not writable — and it has no add-in install tool, no recursive transfer, and
no database export. Conversely the CLI can only move files, so it cannot touch content. SKILL.md has the
full routing table.

Out of scope: `dw query`, `dw database`, `dw swift`, `dw config`. `dw command` appears only as the
last-resort route for deleting or moving archive files that MCP cannot reach. Working on the CLI's own
source is covered by `CLAUDE.md` in the [CLI repository](https://github.com/dynamicweb/CLI).

## Setup

```bash
npm install -g @dynamicweb/cli
```

Requires Node.js >= 20.12.0. Verify with:

```bash
npm ls -g @dynamicweb/cli
```

Then create an API key in the solution's admin, under **Settings → System → Developer → Api Keys**. The
value is `<prefix>.<key>` and the admin list only ever shows it masked, so copy it at creation time.

Store it in a file rather than pasting it into a conversation or a command line:

```bash
printf '%s' 'YOUR_KEY' > ~/.dw-apikey
```

Every command then reads it at call time:

```bash
dw files ./templates Templates -i -o --host <solution>.dynamicweb.cloud --apiKey "$(cat ~/.dw-apikey)"
```

**`dw login` is never used.** It is an interactive user-credential prompt, is unavailable on
`*.dynamicweb.cloud` solutions, and writes a key into `~/.dwc` as a side effect. API key authentication is
the only path — not a fallback for when login fails.

## Usage

This is a **knowledge skill**. There are no scripts to run; it tells Claude Code how to drive the `dw`
binary safely and how to verify the result. See [SKILL.md](SKILL.md) for the full reference.

## Key principles

- Prefer the Dynamicweb MCP server when one is connected; `dw command` / raw `/Admin/Api` is a last resort
- Content (pages, paragraphs, item fields) lives in the database — `dw files` cannot reach it
- Always authenticate with `--host` + `--apiKey`; never `dw login`, not even as a fallback
- State the target host out loud before running anything that writes — a typo in `--host` is the blast radius
- Import is `<local> <remote>`; export is `<remote> <local>` — the positionals swap meaning
- `-o` is required to update an existing file; without it the skip is silent and still exits 0
- On 1.1.2 that skip is completely invisible — `dw files` prints no API response, so always read the state back
- `dw install` reporting success does not prove the add-in loaded; verify against the feature, not the DLL
- Never claim success from the absence of an error — read the state back with `dw files -l`
- `dw install` never restarts the solution: without `-q` the assembly hot-loads immediately, with `-q` it waits for a recycle
- Trigger a recycle deliberately by uploading `recycle.txt` to `System/CloudHosting` (needs `-o`)
- Never accept a key pasted into chat; read it from a file at call time

## Troubleshooting

**Path conversion warning in Git Bash.** Git Bash rewrites leading-slash paths, corrupting remote paths.
Run `export MSYS_NO_PATHCONV=1`, or write remote paths without the leading slash.

**Version-dependent behaviour.** `dw install --output json` and wildcard install paths are present in
1.1.2 and absent in 1.0.16. On 1.0.16 wildcards are actively broken: the upload step uses the raw path
while only activation resolves the glob, so the run dies with an unhandled `ENOENT` on the literal `*`
path and exits 1. It fails loudly and uploads nothing — tested on 1.0.16. Pass fully resolved paths on
1.0.x.

**`dw --version` prints `unknown`** on both 1.0.16 and 1.1.2 — it never identifies the version. Use
`npm ls -g @dynamicweb/cli` instead.

**TLS errors are not certificate problems.** The CLI's HTTPS agent sets `rejectUnauthorized: false` on
purpose so self-signed dev hosts work. A TLS failure points at the host or protocol instead.
