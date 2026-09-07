---
name: dw-cli
description: Use when deploying a built .dll or .nupkg add-in to a Dynamicweb 10 solution, uploading or updating templates and other Files-archive content, exporting a solution's Files archive to disk, triggering a solution recycle, or deciding whether a Dynamicweb change belongs to the `dw` CLI or the Dynamicweb MCP server.
---

You are operating a live Dynamicweb 10 solution through the `dw` CLI. Every command here writes to a real
environment, so the order is always: confirm which environment you are pointed at, act, then verify by
reading back from the solution.

## Scope

This skill covers two workflows, both of which move **files**:

1. **Add-in install loop** — build a custom .NET library, push it to a solution, confirm it loaded.
2. **Files archive** — upload/update files (templates, assets) on a solution, or export them to disk.

**It does not cover changing content.** Pages, paragraphs, item fields, products and users live in the
database, not the Files archive, and belong to the Dynamicweb MCP server — see the next section, which
decides which tool a task belongs to before you run anything.

Also not covered: `dw query`, `dw database`, `dw swift`, `dw config`. `dw command` appears once, only as
the last-resort route for deleting or moving archive files that MCP cannot reach. For working *on* the
CLI's own source, read `CLAUDE.md` in the CLI repository instead.

## Choosing your tool: MCP or the CLI

**Ask one question: am I changing a file, or a record?**

- **A file in the Files archive** — a template, a stylesheet, an asset, an add-in `.dll`, a marker file
  → **`dw` CLI**. This skill.
- **A record in the database** — page, paragraph, item field, product, user, order
  → **Dynamicweb MCP**, if a server is connected.

Everything below is detail on that one split. The two tools barely overlap, so this is a routing
decision, not a preference.

**How to tell which you are dealing with:** if a string is visible on the site but `grep` finds it
nowhere in the Files archive, it is database content — reach for MCP. Page and paragraph text lives in
item fields, not in any `.cshtml`, so no amount of template editing will change it.

### Quick reference

| What you want to change | Use | Why |
|---|---|---|
| A `.cshtml` template, or anything outside `/Files/Images` | `dw files` | MCP cannot write there at all |
| Bulk or recursive archive push/pull | `dw files` | MCP has no recursive or bulk transfer |
| An add-in (`.dll`, `.nupkg`) | `dw install` | MCP has no add-in tool |
| A recycle | `dw files` → `recycle.txt` | MCP cannot write `System/CloudHosting` |
| Database export, Swift release | `dw database`, `dw swift` | MCP has no equivalent |
| Page, paragraph, item field, product, user, order | **MCP** | Typed, validated, read-modify-write |
| Product images, Integration source files | **MCP** | Typed, and inside its writable paths |
| Reading a template just to inspect it | Either | MCP `read_file` is cheaper than an export |
| Anything MCP covers | **MCP**, never `dw command` | See below |
| Nothing above fits and no MCP server exists | `dw command` + `CommandByName` | Last resort |

### Never hand-build an API call when MCP can do it

If a Dynamicweb MCP server is connected, use it rather than `dw command` or a raw `/Admin/Api` POST. MCP
parameters are typed and validated, read-modify-write means a record's untouched fields survive, and
errors come back as errors. A hand-built command POST has none of that: a wrong JSON envelope returns
`status: 'ok'` and changes nothing, and a partial model can overwrite fields you never sent.

### Why MCP cannot replace this CLI

The MCP file tools are restricted by path. Verified against a 10.29 server:

- `upload_file` writes **only** to `/Files/Images` and `/Files/Files/Integration`. Templates and config
  locations are explicitly **not writable** — deploying a `.cshtml` through MCP is impossible.
- `delete_file` and `move_file` are limited to those same two folders.
- `read_file` reads `/Files/Templates` and `/Files/System/Styles`, but rejects binary files.
- `list_files` is **not recursive** and caps at 1000 entries.
- Uploads cap at 25 MB, one file per call.
- There is **no add-in install tool**.

### Before writing through MCP, confirm which solution it points at

An MCP server is configured against one specific solution, and that may not be the `--host` you are
targeting with the CLI. A write can otherwise land on an entirely different environment than the one
under discussion.

**Do not identify the solution by its area domains.** `get_areas` returns each area's configured
`domain`, and that list routinely omits the hostname you are actually using — a solution reached at
`<name>.dynamicweb.cloud` can report only its internal `*.dynamicweb.dk` domains. Judging by that list
alone gives a false negative and will stop you writing to the right solution.

**Fingerprint it instead.** Read back through MCP something whose exact content you already know from the
CLI side — best of all something you just wrote yourself:

```bash
dw files Templates/Designs/Swift-v2/Swift-v2_Master.cshtml ./check -e --host <host> --apiKey "$(cat ~/.dw-apikey)"
```

then `read_file` the same path through MCP and compare. Matching content means one solution. This works
because `/Files/Templates` is readable through MCP even though it is not writable.

## Step 0 — Preflight

Run these before the first `dw` command in a session. Do not skip; most failures in this workflow are
preflight failures that surface later as a confusing API error.

### Check the version — it changes what is available

```bash
npm ls -g @dynamicweb/cli
```

Use this, not `dw --version`: that prints `unknown` on **both** 1.0.16 and 1.1.2, so it is never a way
to tell versions apart.

Behaviour differs by version, and the differences matter. These are the two versions actually verified —
if you are on something in between, confirm with `dw <command> --help` rather than assuming:

| Capability | 1.0.16 | 1.1.2 |
|---|---|---|
| `dw install --output json` (machine-readable envelope, exit code 1 on failure) | absent | present |
| Wildcards in the `dw install` path | **broken — do not use** | works |

The wildcard case on 1.0.16 is a real bug, not just a missing feature: `install.js` passes the raw
`argv.filePath` to the upload step while only the activation step resolves the glob. Tested on 1.0.16, the
upload therefore tries to open a literal `*` path and the process dies with an unhandled
`ENOENT ... PartnerDaysExtended.AddIn*.dll` and a Node stack trace. It fails loudly and exits 1 — nothing
is uploaded and nothing is half-installed. On 1.0.x, always pass a fully resolved path.

To upgrade:

```bash
npm install -g @dynamicweb/cli
```

### Git Bash on Windows

Git Bash rewrites paths that begin with `/`, which corrupts the **remote** paths these commands take.
The CLI prints a warning about this. Export the guard once per shell session:

```bash
export MSYS_NO_PATHCONV=1
```

The check is an exact string comparison against `1`, so `=true` or `=yes` will not disable the warning or
the conversion.

If you would rather not set it, write remote paths without a leading slash (`Templates`, not `/Templates`).
PowerShell and CMD are unaffected.

### Confirm the target solution

Because every command carries an explicit `--host`, the target is whatever you type — there is no ambient
"current environment" to inherit and no `dw env` switching to do. That puts the burden on you:

**State the host out loud before running any command that writes, and confirm it is the solution the user
meant.** A typo in `--host` is the whole blast radius.

`~/.dwc` may exist on the machine from earlier use. Ignore it — `--host` bypasses it entirely, so a stale
environment in that file cannot redirect a command, and its presence is not a reason to skip `--apiKey`.

### Authentication — always an API key, never `dw login`

**Authenticate with `--host` and `--apiKey`. Do not use `dw login`, and do not suggest it.**

This is not a fallback for when login fails, and not specific to cloud hosting — it is the only
authentication path this skill uses. `dw login` runs an interactive user credential prompt, is unavailable
on `*.dynamicweb.cloud` solutions entirely, and writes a key into `~/.dwc` as a side effect. None of that
is wanted.

**No exceptions:**
- Not when `~/.dwc` already has an environment configured — pass `--host`/`--apiKey` anyway
- Not when a command returns 401 — the fix is a valid key, never "try logging in"
- Not "just this once, to get set up"
- Do not run `dw login` yourself, and do not tell the user to run it

If there is no key, ask the user to create one — tell them exactly where:

> **Settings → System → Developer → Api Keys**

A key has a name, a description, an owner, and an expiry date, and its value is `<prefix>.<key>` — the
admin list only ever shows it masked, so it must be copied at creation time. Pass it per command:

```bash
dw files <args> --host <solution>.dynamicweb.cloud --apiKey "$DW_API_KEY"
```

`--host` bypasses the environment config entirely (`setupEnv` stops looking at `~/.dwc`) and `--apiKey`
bypasses login. `--host` is rejected without `--apiKey`. Add `--protocol http` only for a local HTTP host.

> **Security bug: `dw query` and `dw command` leak the API key into the request URL.** Both build their
> request parameters as `Object.keys(argv).filter(k => !exclude.includes(k))`, and their `exclude` lists
> contain `apiKey` but not the kebab-case alias `api-key` that yargs also puts in `argv`. So every
> `dw query` / `dw command` call made with `--apiKey` appends `&api-key=<your key>` to the URL -- where it
> reaches server logs and proxies -- and the CLI prints the whole URL, key included, in its error output.
> Verified on 1.1.2. `dw files` and `dw install` are unaffected. Prefer reading the log through the API
> with your own request (Authorization header) over `dw query` until this is fixed, and rotate any key
> that has been used this way.

**Never accept an API key pasted into the conversation, and never write one into a command line
literally.** Have the user put it in a file and read it at call time, so the value stays out of the
transcript and out of shell history:

```bash
dw files ./templates Templates -i -o --host example.dynamicweb.cloud --apiKey "$(cat ~/.dw-apikey)"
```

Do not "simplify" this by writing the key into `~/.dwc` so later commands can omit `--apiKey`. That
trades an explicit, auditable target for an ambient one, and it is the same implicit-environment problem
that `--host`/`--apiKey` exists to avoid.

## Step 1 — Add-in install loop

```bash
dotnet build -c Release
```

```bash
dw install ./bin/Release/net10.0/YourProject.dll
```

`dw install` accepts `.dll` and `.nupkg`. It does two things: uploads the file to the solution's
`System/AddIns/Local` folder (always overwriting), then calls the `AddinInstall` API to activate it.

### Flags

- `-q`, `--queue` — queue activation for the next Dynamicweb recycle instead of applying it now.
- `--output json` — structured envelope on stdout, exit code 1 on failure (**1.1.2; absent in 1.0.16**).
- `-v`, `--verbose` — log the resolved path.

### Choosing between immediate and queued

**Neither variant recycles the solution.** `dw install` never restarts the application, so the choice is
not about avoiding downtime:

- **Without `-q`** — the assembly is loaded into the *running* application straight away. No restart, no
  dropped requests. This is what makes the dev loop fast.
- **With `-q`** — activation is deferred until the next recycle, so the new code does not go live until
  a recycle happens.

So `-q` is about *when the new code takes effect*, not about protecting the site from a restart. Pick on
that basis:

- Local or dev environment: omit `-q`. Immediate load is the point.
- Shared, test or production: use `-q` when you do not want new code becoming live at an unannounced
  moment, then activate deliberately at a planned recycle. Say plainly that the install is staged.

Do not describe a non-queued install as "forcing a recycle" — it is not one. The CLI's own flag text
("Queues the install for next Dynamicweb recycle") describes only the deferred case and is easy to
over-read in the other direction.

### Triggering a recycle

On a cloud-hosted solution there is no `dw` command that restarts the application. You trigger a recycle
by dropping a marker file named `recycle.txt` into `/Files/System/CloudHosting` — the same folder that
already holds the platform's other control files (`ChangeVersion-Readme.txt`, `BackupRestoreDB/`):

```bash
echo recycle > ./recycle.txt
dw files ./recycle.txt System/CloudHosting -i -o --host <host> --apiKey "$(cat ~/.dw-apikey)"
```

This is the companion to `dw install -q`: queue the add-in, then recycle when the disruption is
acceptable.

**The platform consumes the marker.** Tested: `recycle.txt` was absent before the upload and absent
again afterwards — it is deleted once the recycle is triggered. So it never accumulates, and `-o` is not
actually required for this to work a second time. Passing `-o` anyway is harmless and saves you from
depending on that cleanup, which is why it is in the command above.

**Verified: this really does restart the solution.** During the recycle the site returned HTTP 503, then
came back at 200 after a short window. In-flight requests are dropped. Do not trigger one on a shared
environment without asking.

> **`dw files` breaks while the solution is down.** Any `dw files` call during the recycle window dies
> with `SyntaxError: Unexpected token '<', "<!DOCTYPE "... is not valid JSON` — `getFilesStructure` calls
> `res.json()` on the HTML error page the platform serves at 503. It is not a CLI bug you can work
> around; just wait for the site to answer 200 and retry.

**Templates do not need a recycle.** A `.cshtml` pushed with `dw files` renders on the very next request
— verified by pushing a change to `Swift-v2_Master.cshtml` and seeing it live immediately, with no
recycle and no cache clear. Recycles are for add-in assemblies, not for templates or other Files content.
Do not restart a solution just because a template edit "hasn't shown up"; check `-o` and the `model` array
first, since a silently skipped upload looks exactly like a caching problem.

### Asserting success

Where the envelope exists, prefer it — it is the only reliable signal:

```bash
dw install ./bin/Release/net10.0/YourProject.dll --output json
```

`ok: true` plus exit code 0 means uploaded and activated. On failure `ok` is `false`, `errors[]` carries
the API detail, and the process exits 1.

Verified on 1.1.2, a failure gives a clean envelope and no crash:

```json
{ "ok": false, "status": 1, "data": [],
  "errors": [ { "message": "Could not find any files with the name ./bin/.../DoesNotExist.dll" } ] }
```

exit 1. A success carries two `data` entries — `type: "upload"` and `type: "install"` (the latter with
`"message": "App(s) successfully installed"`) — plus `meta.resolvedPath`, `meta.queued` and
`meta.filesProcessed`.

**On 1.0.16 the same bad path behaves completely differently:** there is no envelope, and the upload step
opens the raw path before anything validates it, so the process dies with an unhandled
`ENOENT ... open '<path>'` and a Node stack trace. It still exits 1, but you never see
`Could not find any files with the name ...` — that message is unreachable from `dw install` on 1.0.16.
Another reason to be on 1.1.2 in any automated context.

On 1.0.x there is no envelope. Check the exit code and read the human output. Verified on 1.0.16, a
successful queued install prints the upload response with the target path, then the activation line:

```text
model: [ '/Files/System/AddIns/Local/YourProject.dll' ]
Installing addin
Addin installed
```

and exits 0; a missing file exits 1. Then verify via Step 3.

### A successful install does not prove the add-in loaded

**`ok: true` and "App(s) successfully installed" only mean the file was uploaded and the API accepted the
registration call. They are not evidence that the assembly loaded or that its types are usable.**

Tested on a live 10.29 solution with a purpose-built `BaseScheduledTaskAddIn`:

- the upload landed (`model: [ '/Files/System/AddIns/Local/...dll' ]`), the DLL persisted in the archive
- the install returned `ok: true`, `status: 200`, `"App(s) successfully installed"`, exit 0
- **the add-in type never appeared** in the solution's scheduled-task type list — not after the
  non-queued install, and not after a full recycle either

**The cause is now established: the assembly was built against newer packages than the solution runs.**
An A/B test with identical source settled it — compiled against `Dynamicweb` 10.28.9 (stable) the add-in
loaded and its type appeared; recompiled against `Dynamicweb.Suite 10.29.2-PreRelease` the same code
stopped loading and the type disappeared again. `dw install` reported success identically both times.

So when an installed add-in is missing, check the package versions it was built against before suspecting
the CLI or the upload. `Version="10.*"` (latest stable) is the safe default; a prerelease or a version
newer than the host loads nothing and says nothing.

**So verify against the feature, not the command.** Confirming the DLL is in `System/AddIns/Local` proves
only the upload. Confirm the add-in itself is present — its type in the relevant admin list, its provider
selectable, its task creatable — and if it is missing, suspect the assembly's package versions against
the solution's runtime before suspecting the CLI.

### Wildcards (1.1.2; do not use on 1.0.16)

The path is glob-matched against the directory and the **first** match wins, which is useful when the
filename carries a version:

```bash
dw install "./bin/Release/net10.0/YourProject*.dll"
```

Quote the pattern so the shell does not expand it first. If more than one file matches you get whichever
the directory listing returns first.

Verified on 1.1.2. You do not have to guess which file it picked — `--output json` reports it:

```json
"meta": { "resolvedPath": "C:\...\net10.0\YourProject.dll", "filesProcessed": 1 }
```

Read `meta.resolvedPath` whenever the pattern could match more than one file.

## Step 2 — Files archive

One command, `dw files [dirPath] [outPath]`, does listing, export, and import.

> **The two positionals swap meaning between export and import.** This is the single most common mistake.
>
> - **Export (`-e`)** — `dirPath` is **remote**, `outPath` is **local**. Pulls *from* the solution.
> - **Import (`-i`)** — `dirPath` is **local**, `outPath` is **remote**. Pushes *to* the solution.
>
> Getting it backwards on an import means you name a local directory as the destination, and the upload
> goes somewhere unintended on the solution.

### Uploading and updating files (import)

Push one local folder's files into `/Templates` on the solution:

```bash
dw files ./templates Templates -i -o
```

Push a whole local tree, subdirectories included:

```bash
dw files ./templates Templates -i -r -o
```

Update one specific file — `dirPath` may be a file rather than a directory, and `outPath` stays the remote
*directory* it belongs in:

```bash
dw files ./templates/YourFile.cshtml Templates -i -o
```

> **`-o` / `--overwrite` is required to update anything that already exists.** Without it the CLI sends
> `skipExistingFiles=true` and the server silently skips every existing file. The command still prints
> `status: 'ok'`, still says `Finished uploading files. Total files: 1`, and still exits 0 —
> **nothing changed**. If the task is "update a file on the solution", `-o` is mandatory.

**On 1.0.16 there is a tell; on 1.1.2 there is none.** This is the one place where the newer version is
worse, and it is tested on both.

1.0.16 prints the API response, so a skip is visible immediately:

```text
model: [ '/Files/_dwcli-test/probe.txt' ]   <- written
model: []                                   <- skipped, despite "Total files: 1" and exit 0
```

1.1.2 prints **no API response at all** for `dw files`. A skipped import looks exactly like a successful
one:

```text
Uploading files
Uploading chunk 1 of 1
Finished uploading files. Total files: 1, total chunks: 1
```

That is the output whether the file was written or silently skipped, and the exit code is 0 either way.
The cause is structural: `uploadFiles` now routes the response through `output.addData()`, which is a
no-op unless `--output json` is set — and `dw files` has no `--output json` on any version.

**Consequence: on 1.1.2 the only way to know an import landed is to read the state back** (Step 3). Do not
skip it, and do not treat "Total files: 1" as confirmation of anything.

Other import flags:

- `-r`, `--recursive` — walk subdirectories. Without it, only files directly in `dirPath` are sent.
- `--createEmpty` — sets `createEmptyFiles=true` on the upload call. Per the flag's own description, an
  empty file is otherwise not created at the destination.

Behaviour worth knowing:

- Missing remote directories are created automatically.
- Large pushes are chunked automatically, so pushing a whole template tree in one call is fine.
- **`dw files` has no `--output json` on any version.** The response body is console-printed, not a
  structured envelope, and the exit code is 0 whether or not anything was written. Read the `model` array
  as the immediate signal, and confirm anything that matters with a listing or diff (Step 3).

### Deleting and moving: not `dw files`, but not impossible

`dw files` only lists, downloads, and uploads — there is no delete, rename, or move flag on either
version. Uploading a renamed copy therefore leaves the original in place and silently doubles the file.

The capability does exist through the Management API via `dw command`. **MCP does not rescue you here
for most paths:** its `delete_file` and `move_file` only work under `/Files/Images` and
`/Files/Files/Integration`, so deleting or moving a *template* is one of the few cases where
`dw command` is the only route, not a fallback. Use MCP's file tools when the target is inside those two
folders; otherwise use the commands below. Verified present on a 10.29 solution:

| Command | JSON model |
|---|---|
| `FileDelete` | `{ "DirectoryPath": "...", "Ids": ["..."] }` |
| `DirectoryDelete` | `{ "Path": "..." }` |
| `FileMove` | (query `CommandByName` for the current shape) |

Verified working call — note the body is **bare, with no `model` wrapper**:

```bash
dw command FileDelete --json '{"DirectoryPath":"_scratch","Ids":["_scratch/probe.txt"]}' --host <host> --apiKey "$(cat ~/.dw-apikey)"
```

Returns `status: 'ok'` and the file is gone from the next listing.

**The JSON envelope is not consistent across commands.** These file commands take the model's fields at
the top level, exactly as `dw files` posts them to `FileDownload`. Other commands do not: the CLI README
shows `PageCopy` wrapped as `{"model":{...}}` and `PageDelete` bare as `{"id":"..."}`. Do not assume a
wrapper. Mirror how `dw files` calls the same family of endpoints where you can, otherwise try bare first
and read the response — a wrong envelope comes back `ok` with nothing changed, the same silent-success
failure mode as a missing `-o`.

**Discovering a command's parameters:** `dw command <name> -l` does not work — `getProperties()` returns
the string `"This option currently doesn't work"` before it reaches the API call, so the README's `-l`
documentation is wrong. Query the endpoint directly instead; it is a GET, so it is safe to probe:

```bash
curl -s -H "Authorization: Bearer $KEY" "https://<host>/Admin/Api/CommandByName?name=FileDelete"
```

A real command returns HTTP 200 with a `model` field holding the JSON shape; an unknown name returns 500.
That request/response pair is the reliable way to build any `dw command` call.

### Exporting files (export)

Pull `/Templates` from the solution into `./templates`, files included, recursive:

```bash
dw files Templates ./templates -f -r -e
```

Pull a single file:

```bash
dw files Templates/Translations.xml ./templates -e
```

Export flags:

- `-r`, `--recursive` — include subdirectories.
- `-f`, `--includeFiles` — include files, not just the directory skeleton.
- `--raw` — keep the downloaded `.zip` instead of unpacking it.
- `--iamstupid` — also export `system/log` and `.cache`. Effectively never wanted; these are huge.

Whether a path is treated as a file or a directory is inferred from whether it has an extension. Override
when that inference is wrong:

- `-ad`, `--asDirectory` — a directory whose name contains a dot (`templates/templates.v1`).
- `-af`, `--asFile` — a file with no extension (`templates/testfile`).

Running `-e` with no `dirPath` triggers a full-archive export behind an interactive confirmation. Do not
do this unless the user asked for it.

## Step 3 — Verify (mandatory)

`dw files` cannot report failure structurally, and on 1.0.x neither can `dw install`. Never report success
from the absence of an error. Read the state back.

Confirm files landed, including contents, recursively:

```bash
dw files Templates -l -f -r
```

Confirm an add-in binary reached the solution:

```bash
dw files System/AddIns/Local -l -f
```

Check that the filenames you pushed are actually present. For an update, remember that a missing `-o`
produces a clean-looking run with no change — so if the listing shows the file but you are not sure the
*content* updated, export it back and diff:

```bash
dw files Templates/YourFile.cshtml ./verify -e
```

```bash
diff ./verify/YourFile.cshtml ./templates/YourFile.cshtml
```

Only claim the change is live after a listing or diff confirms it.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Warning about path conversion; remote path lands in the wrong place | Git Bash rewriting `/Templates` | `export MSYS_NO_PATHCONV=1`, or drop the leading slash |
| Import runs clean, exits 0, but the file is unchanged | `-o` omitted, so the server skipped the existing file | Re-run with `-o` |
| Import wrote to an unexpected remote location | Positionals swapped | Import is `<local> <remote>`; export is `<remote> <local>` |
| `dw install` cannot find the file | The path does not exist on disk (or is a wildcard on 1.0.16) | Check the build output path and target framework folder. Both versions exit 1, but they report it very differently — see below |
| Install uploads but the add-in never appears | Wildcard path on 1.0.16 | Pass a fully resolved path, or upgrade |
| 401 / unauthorized | API key missing, expired, or revoked | Ask the user for a new key from **Settings → System → Developer → Api Keys**. Never fall back to `dw login` |
| Add-in installed but the solution still runs old code | Activation was queued with `-q`, so it waits for a recycle | Trigger a recycle by uploading `recycle.txt` to `System/CloudHosting` (with `-o`), or re-run the install without `-q` to hot-load it into the running application |
| Empty local files never arrive | Zero-byte files skipped by default | Add `--createEmpty` |
| `SyntaxError: Unexpected token '<', "<!DOCTYPE "... is not valid JSON` | The solution served an HTML error page (usually 503 mid-recycle) and the CLI called `res.json()` on it | Wait until the site answers 200, then retry. Not fixable from the command line |
| Import prints `Total files: 1` on 1.1.2 but nothing changed | 1.1.2 prints no API response for `dw files`, so a missing-`-o` skip is invisible | Never trust that line. Read the state back with a listing or an export-and-diff |
| `dw install` says `App(s) successfully installed` but the add-in is unusable | The upload and registration succeeded; the assembly still may not have loaded | Check the add-in's own admin list, not the DLL. Suspect package versions built against the solution's runtime |

Self-signed certificates are accepted by design — the CLI's HTTPS agent sets `rejectUnauthorized: false`
so local dev hosts work. A TLS error therefore points at the host or protocol, not the certificate.

## What is verified, and what is not

Claims here do not all carry the same weight. Treat the last group with suspicion and check before
relying on it.

**Verified live against a 10.29 cloud solution (2026-09-04):** API-key auth through
`--host`/`--apiKey`; `dw files` list, import and export; the missing-`-o` silent skip and the `model: []`
tell; automatic creation of missing remote directories; a `.cshtml` deploy taking effect with no recycle;
`FileDelete` with a bare body; reading `/Files/Templates` through MCP; MCP `set_paragraph_item_fields`
leaving sibling fields intact.

**Read from the CLI source but never executed:** everything version-specific in the 1.0.16/1.1.2 table —
only 1.0.16 was installed, so `--output json` and the wildcard behaviour were established by reading
`install.js`, not by running it. Same for the `dw command -l` dead-code bug.

**Reported by the solution owner, not tested:** the `recycle.txt` mechanism in `System/CloudHosting`.
The `-o` consequence described there is inferred from verified upload behaviour, not observed on the
recycle path. Likewise the `-q` semantics — that a non-queued install hot-loads the assembly into the
running application and that neither variant restarts it. An earlier draft of this skill wrongly claimed
a non-queued install recycles the solution; do not reintroduce that.

**Tested on both 1.0.16 and 1.1.2:** the add-in install loop — a purpose-built class library was
compiled and installed queued (`-q`) against a live solution on each version. Confirmed: upload target
`System/AddIns/Local`, the `model` path, `Addin installed`, exit 0 on success and 1 on a bad path, the
1.0.16 wildcard crash, and on 1.1.2 both working wildcards (with `meta.resolvedPath`) and the
`--output json` envelope on success and failure.

**Tested and found NOT to work as expected:** a non-queued install did not make the add-in's type
available, and neither did a subsequent full recycle. `dw install` reporting success is therefore not
evidence the add-in works. Root cause unresolved — see the section above.

**Verified live:** the `recycle.txt` marker genuinely recycles the solution (503 during, 200 after), the
platform deletes the marker afterwards, and `dw files` crashes on the HTML error page served mid-recycle.

**Still unverified:** whether an add-in built against matching (non-prerelease) package versions loads
correctly. That is the open question left by the failed probe.

## Source of truth

The published README is behind the source and omits several of the flags above. When something here
disagrees with observed behaviour, read the command source and trust that:

- `bin/commands/files.js` — list, export, import, upload semantics
- `bin/commands/install.js` — add-in upload and activation
- `bin/commands/env.js`, `bin/commands/login.js` — environment and auth resolution

Do **not** trust `CLAUDE.md` in the CLI repository as a flag reference. As of 1.1.2 it documents an
OAuth mode (`--auth`, `--clientId`, `--clientSecret`) and a global `--output json` that exist nowhere in
`bin/`. The only global flags are `--verbose`, `--protocol`, `--host`, `--apiKey`, and `--output json` is
specific to `install`.

Repository: https://github.com/dynamicweb/CLI

Docs: https://doc.dynamicweb.dev/documentation/fundamentals/code/CLI.html
