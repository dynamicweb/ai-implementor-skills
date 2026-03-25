# Dynamicweb Extension Skill

Knowledge base for building best-practice, upgrade-safe extensions for Dynamicweb and Dynamicweb Swift.

## What this skill covers

- **NotificationSubscribers** — hook into platform events (order placed, user login, page render)
- **Providers** — replace core subsystems (price, tax, shipping, stock, payment, feed)
- **ScheduledTask AddIns** — run logic on a timer or via the admin task runner
- **UpdateProviders** — schema migrations for custom database tables
- **Database queries** — raw SQL via `Dynamicweb.Database`
- **Services** — accessing platform data from any extension type
- **ViewModel extensions** — add custom data to Razor template contexts
- **Swift CSS customization** — override variables and selectors in `custom.css`
- **Razor template overrides** — override Swift templates without editing core files
- **Custom ItemTypes** — new content building blocks for the admin paragraph picker

## Usage

This is a **knowledge skill** used by Claude Code when assisting with Dynamicweb extension development. See [SKILL.md](SKILL.md) for the full reference.

There are no scripts to run. The skill provides guidance and code scaffolding via Claude.

## Key principles

- Never edit Swift core files — all overrides go in a separate customization layer
- Choose the least invasive approach: CSS variables → CSS selectors → Razor overrides → new ItemTypes
- Prefer `Services.*` over raw SQL for data that services already expose
- All custom database tables must use an `UpdateProvider` for idempotent schema migrations
- Prefix custom table names and ItemType system names with your project/company name
