# Dynamicweb Admin UI Skill

Extending the Dynamicweb 10 administration interface from a customer solution — from requirement to a
change verified working on the solution.

## What this skill covers

- **Routing the task** — own screen vs. screen injector vs. action-menu item vs. custom `ScreenType`
- **The four parts of a screen** — DataModel, Query, Command, Screen, with compile-verified skeletons
- **Making it reachable** — your own area, a section in its tree, and nodes pointing at your screens
- **Extending screens you do not own** — `ScreenInjector<TScreen>` and its specialised bases
- **Deploy and verification** — `dw install`, and how to prove the change actually took effect

It deliberately does **not** repeat Dynamicweb's own
[Custom UI documentation](https://doc.dynamicweb.dev/documentation/extending/administration-ui/index.html),
which explains the concepts well and stays current. This skill is the operational layer around it.

## Relationship to the other skills

| Skill | Role |
|---|---|
| `dw-extend-admin-ui` | admin interface extensions — screens, areas, injectors |
| `dw-extend` | other extension types — NotificationSubscribers, Providers, ScheduledTasks, UpdateProviders, Swift frontend |
| `dw-setup-cli` | moving the built assembly onto a solution and verifying it landed |
| `azure-devops` | the surrounding work-item, branch and PR workflow |

## Setup

One package reference is enough — it transitively pulls `Dynamicweb.CoreUI`, `Dynamicweb.Core` and the
rest:

```xml
<PackageReference Include="Dynamicweb.Application.UI" Version="10.*" />
```

Requires .NET SDK 10.0. Deploying needs the `dw` CLI and an API key — see the `dw-setup-cli` skill.

## Usage

This is a **knowledge skill**. There are no scripts to run; it tells Claude Code which building block a
task calls for, gives skeletons that compile, and insists on verifying the result in the admin. See
[SKILL.md](SKILL.md) for the full reference.

## Key principles

- Decide first whether you own the screen — subclassing a core screen compiles, deploys, and never renders
- Never build against newer packages than the solution runs; a too-new assembly fails to load silently
- `AddInManager` discovers everything automatically, so nothing reports a discovery failure
- `dw install` reporting success is not evidence your screen works — open the admin and look
- Install without `-q`, or the change waits for the next recycle
- Injectors run inside someone else's screen: check the component shape and the model for null before touching them

## Troubleshooting

**Nothing appears and there is no error.** Read `/Files/System/Log/AddInManager/TypeLoadErrors.log`. A
version mismatch shows up there as a `ReflectionTypeLoadException` naming the assembly it could not find.

**`CS0311` on a `NavigationSection<>`.** `AreaBase` is in `Dynamicweb.CoreUI.Application`, not
`Dynamicweb.CoreUI.Navigation`. Without that using, your area type has no base class and the generic
constraint fails with a misleading message.

**An icon name will not compile.** `Icon` members are a fixed set; `Icon.List` does not exist. Complete
them from the type rather than guessing.

**Runtime errors during rendering** are written to the `GeneralLog` table, not to a file. The dw-extend
skill's debugging section has the query and filters.
