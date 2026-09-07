---
name: dw-admin-ui
description: Use when extending the Dynamicweb 10 administration interface from a customer solution — adding a screen, an area in the sidebar, a node in an area tree, an action-menu item, or changing a screen you do not own — and when taking such a change from requirement to built, installed and verified on a solution.
---

You are extending the Dynamicweb 10 admin interface (the "backend") from your own assembly in a customer
solution. This skill owns the route from requirement to a change **verified working on the solution**.
Committing and raising a PR belong to the surrounding work-item workflow, not here.

Dynamicweb's own documentation explains the concepts well and is kept current — this skill does not repeat
it. Read [Custom UI](https://doc.dynamicweb.dev/documentation/extending/administration-ui/index.html) and
[Screen architecture](https://doc.dynamicweb.dev/documentation/extending/administration-ui/screenconcept.html)
for the model. What follows is the operational layer: which building block to reach for, skeletons that
compile, and how to prove the change actually took effect.

## Step 1 — Route the task before writing anything

**One question decides the whole approach: do you own the screen?**

| What you need | Building block | Why |
|---|---|---|
| A screen for **your own** data | Subclass a screen type + your own area/nodes | You control the screen, so build it |
| Add fields, columns or widgets to a **Dynamicweb or third-party** screen | **Screen injector** | You cannot subclass a core screen and have Dynamicweb use yours instead |
| Only an extra action on an existing screen | Action-menu item | Lighter than an injector |
| A layout the built-in screen types cannot express | Custom `ScreenType` with your own Razor views | Last resort — most work, most to maintain |

Getting this wrong is expensive: subclassing a core screen compiles, deploys, and then simply never
renders, because nothing routes to your subclass. If the screen belongs to someone else, it is an
injector.

**Which screen type**, once you know it is yours:

| Screen type | Use for |
|---|---|
| `ListScreenBase<TModel>` | browsing a collection in a table |
| `EditScreenBase<...>` | editing one record through a tabbed form |
| `OverviewScreenBase<TModel>` | a summary of one record, assembled from widgets |

`GridEditScreen` and `PromptScreen` also exist — see the
[screen types](https://doc.dynamicweb.dev/documentation/extending/administration-ui/screentypes/listscreen.html)
docs.

## Step 2 — Project setup

One package reference is enough. `Dynamicweb.Application.UI` transitively brings `Dynamicweb.CoreUI`,
`Dynamicweb.Core`, `Dynamicweb.Ecommerce` and the rest — ten packages in total.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Dynamicweb.Application.UI" Version="10.*" />
  </ItemGroup>
</Project>
```

> **Never build against a newer version than the solution runs.** `Version="10.*"` resolves to the latest
> *stable* and excludes prereleases, which is why it is the right default — do not "helpfully" change it
> to `10.*-*`. An assembly built against newer Dynamicweb assemblies than the host has **fails to load
> silently**: `dw install` still reports success, and nothing appears in the admin. See the dw-cli skill.

## Step 3 — The four parts of a screen

Every screen you build is the same four pieces. All skeletons below are compile-verified against
`Dynamicweb.Application.UI` 10.28.9.

**The model** — one property per column or field. `[ConfigurableProperty]` is what makes it addressable
by the UI.

```csharp
using System.Globalization;
using Dynamicweb.CoreUI.Data;
using Dynamicweb.CoreUI.Data.Validation;

public sealed class WidgetDataModel : DataViewModelBase, IIdentifiable
{
    [ConfigurableProperty]
    public int Id { get; set; }

    [ConfigurableProperty]
    [Required]
    public string Name { get; set; } = "";

    [ConfigurableProperty]
    public int Stock { get; set; }

    public string GetId() => Id.ToString(CultureInfo.InvariantCulture);
}
```

> **Implement `IIdentifiable` from the start**, even for a list-only screen. A plain `DataViewModelBase`
> is enough for a list, but the moment you add an edit or overview screen you need a by-id query, and
> `DataQueryIdentifiableModelBase<TModel, TKey>` constrains `TModel` to `IIdentifiable`. Adding it later
> means touching the model, its queries and its screens at once.
>
> **How identity travels.** `IIdentifiable` is one half of a round trip: the model serialises its identity
> to a *string* the UI can carry in a URL or a list row, and the query parses it back into a typed key.
>
> ```text
> model.GetId() → "42" → query.ModelIdentifier → TryParseIdentifier → SetKey(42)
> ```
>
> `GetId()` and `SetKey()` must therefore be inverses. If `GetId()` produces something
> `TryParseIdentifier` cannot parse into `TKey`, the framework simply does not call `SetKey` — no
> exception — so the query keeps its default key and loads the wrong record or none. For a composite
> identity (`"shopId:productId"`), override the `virtual TryParseIdentifier`; `OnGetData` is
> `internal sealed` and cannot be replaced.

**The query** — reads source data and maps it to the model. The second type parameter is the *source*
type, not the model.

```csharp
using System.Collections.Generic;
using System.Linq;
using Dynamicweb.CoreUI.Data;

public sealed class WidgetsQuery : DataQueryListBase<WidgetDataModel, string>
{
    protected override IEnumerable<string> GetListItems()
        => new[] { "Left-handed widget", "Right-handed widget" };

    protected override IEnumerable<WidgetDataModel> MapModels(IEnumerable<string> items)
        => items.Select(x => new WidgetDataModel { Name = x, Stock = x.Length });
}
```

**The command** — anything that writes. Validate with attributes; return a `CommandResult`.

```csharp
using Dynamicweb.CoreUI.Data;
using Dynamicweb.CoreUI.Data.Validation;

public sealed class WidgetDeleteCommand : CommandBase
{
    [Required]
    public int Id { get; set; }

    public override CommandResult Handle()
        => new() { Status = CommandResult.ResultType.Ok };
}
```

**The screen** — what renders in the workspace.

```csharp
using System.Collections.Generic;
using Dynamicweb.CoreUI.Data;
using Dynamicweb.CoreUI.Lists;
using Dynamicweb.CoreUI.Lists.ViewMappings;
using Dynamicweb.CoreUI.Screens;

public sealed class WidgetListScreen : ListScreenBase<WidgetDataModel>
{
    protected override string GetScreenName() => "Widgets";

    protected override IEnumerable<ListViewMapping> GetViewMappings() => new ListViewMapping[]
    {
        new RowViewMapping
        {
            Columns = new List<ModelMapping>
            {
                CreateMapping(m => m.Name),
                CreateMapping(m => m.Stock),
            }
        }
    };
}
```

### A query for one record

Edit and overview screens are fed by a single-model query, not a list query.

```csharp
using System.Linq;
using Dynamicweb.CoreUI.Data;

public sealed class WidgetByIdQuery : DataQueryIdentifiableModelBase<WidgetDataModel, int>
{
    public int Id { get; set; }

    public override WidgetDataModel? GetModel()
        => WidgetStore.Find(Id) is { } w
            ? new WidgetDataModel { Id = w.Id, Name = w.Name, Stock = w.Stock }
            : null;

    protected override void SetKey(int key) => Id = key;
}
```

### The edit screen

`EditScreenBase<TModel>` needs three things: a name, a save command, and a layout. Note the save command
is `CommandBase<TModel>` — the **generic** base, not the plain `CommandBase` a list action uses.

```csharp
using System.Collections.Generic;
using Dynamicweb.CoreUI.Data;
using Dynamicweb.CoreUI.Screens;

public sealed class WidgetEditScreen : EditScreenBase<WidgetDataModel>
{
    protected override string GetScreenName() => "Edit widget";

    protected override CommandBase<WidgetDataModel>? GetSaveCommand() => new WidgetSaveCommand();

    protected override void BuildEditScreen()
    {
        AddComponents("General", new LayoutWrapper[]
        {
            new LayoutWrapper("Details",
            [
                EditorFor(m => m.Name),
                EditorFor(m => m.Stock),
            ])
        });
    }

    // Optional: per-field behaviour, e.g. make a field read-only
    protected override IEnumerable<EditorMapping> GetEditorMappings() => new List<EditorMapping>
    {
        CreateMapping(m => m.Id) with { ReadOnlyPredicate = m => true }
    };
}
```

The matching save command reads the posted model with `GetModel()`:

```csharp
public sealed class WidgetSaveCommand : CommandBase<WidgetDataModel>
{
    public override CommandResult Handle()
    {
        var model = GetModel();
        if (model is null)
            return new CommandResult { Status = CommandResult.ResultType.Invalid, Message = "No model" };

        WidgetStore.Save(model.Id, model.Name, model.Stock);
        return new CommandResult { Status = CommandResult.ResultType.Ok, Message = "Widget saved" };
    }
}
```

`CommandResult.Message` is shown to the user as a toast, so write it for them. Only fields you added an
`EditorFor` for are rendered; `[Required]` on the model surfaces as the field's required marker.

### The overview screen

`OverviewScreenBase<TModel>` assembles widgets around one record. `Model` is the record the bound query
returned.

```csharp
using Dynamicweb.CoreUI.Displays.Widgets;
using Dynamicweb.CoreUI.Layout;
using Dynamicweb.CoreUI.Screens;

public sealed class WidgetOverviewScreen : OverviewScreenBase<WidgetDataModel>
{
    protected override string GetScreenName() => "Widget overview";

    protected override void BuildOverviewScreen()
    {
        AddWidget(new Widget
        {
            Label = Model?.Name ?? "Widget",
            Component = new ListDisplay<WidgetListScreen, WidgetDataModel>(new WidgetsQuery())
            {
                EnableEditing = false,
                ListNavigateAction = null
            }
        }, Group.GroupWidth.Col_12);
    }
}
```

A widget's `Component` can be a list, a graph, an info card, and its `ContextMenu` can carry actions —
`new ContextMenu().WithActionNode(ActionBuilder.Edit<WidgetEditScreen>(new WidgetByIdQuery { Id = id }))`
is how Dynamicweb's own overview screens link to their edit screen.

### The list screen is the hub

This is the shape Dynamicweb's own screens use, and getting it wrong produces an odd UI. **One tree node
points at the list screen; the edit and overview screens hang off the list**, not off nodes of their own.
Three overrides on `ListScreenBase<T>` do it:

```csharp
// Row click -> the overview for that record
protected override ActionBase? GetListItemPrimaryAction(WidgetDataModel model)
{
    ArgumentNullException.ThrowIfNull(model, nameof(model));
    return NavigateScreenAction.To<WidgetOverviewScreen>().With(new WidgetByIdQuery { Id = model.Id });
}

// Per-row "..." menu -> edit and delete
protected override IEnumerable<ActionGroup>? GetListItemContextActions(WidgetDataModel model)
{
    var query = new WidgetByIdQuery { Id = model.Id };
    return new ActionGroup[]
    {
        new()
        {
            Nodes =
            [
                ActionBuilder.Edit<WidgetEditScreen>(query),
                ActionBuilder.Delete(
                    new WidgetDeleteCommand { Id = model.Id },
                    "Delete widget?",
                    $"Do you want to delete '{model.Name}'?")
            ]
        }
    };
}

// The "+" affordance -> usually a slide-over create screen
protected override ActionNode GetItemCreateAction() => new()
{
    Icon = Icon.Plus,
    Name = "New widget",
    NodeAction = OpenSlideOverAction.To<WidgetCreateScreen>()
};
```

`ActionBuilder` lives in `Dynamicweb.Application.UI.Helpers` — not in the `CoreUI.Actions` namespaces
where the rest of this belongs. `ActionBuilder.Delete` wraps itself in a confirm dialog for you.

Verified on a live solution: one node → list; row click → that record's overview; the row's `...` menu
showing **Edit** and **Delete**; and Save on the edit screen writing through the save command and showing
its `CommandResult.Message` as a toast.

**Three naming rules, all enforced at runtime rather than by the compiler:**

| Base type | Class name must end in |
|---|---|
| `AreaBase` | `Area` |
| `ActionBase` | `Action` |
| `DataViewModelBase` | `Model` |

## Step 4 — Making it reachable

A screen nobody can navigate to is invisible. Give it an area, a section and a node.

> **Namespace trap:** `AreaBase` lives in `Dynamicweb.CoreUI.Application`, while everything else here is
> in `Dynamicweb.CoreUI.Navigation`. Miss that one using and the compiler reports a confusing generic
> constraint failure on your section (`CS0311`) rather than a missing type.

```csharp
using System.Collections.Generic;
using System.Linq;
using Dynamicweb.CoreUI.Actions.Implementations;
using Dynamicweb.CoreUI.Application;   // AreaBase
using Dynamicweb.CoreUI.Icons;
using Dynamicweb.CoreUI.Navigation;

// The class name MUST end in "Area" — AreaBase's constructor throws
// InvalidOperationException otherwise. It is a runtime failure, not a compile error.
public sealed class AcmeArea : AreaBase
{
    public AcmeArea()
    {
        Name = "Acme";
        Icon = Icon.Truck;
        Sort = 90;              // after the built-in areas
    }
}

public sealed class WidgetSection : NavigationSection<AcmeArea>
{
    public WidgetSection(NavigationContext context) : base(context)
    {
        Name = "Widgets";
        Sort = 10;
    }
}

public sealed class WidgetNodeProvider : NavigationNodeProvider<WidgetSection>
{
    public override IEnumerable<NavigationNode> GetRootNodes()
    {
        yield return new NavigationNode
        {
            Name = "All widgets",
            Id = "Acme_Widgets",            // stable, unique
            Icon = Icon.Folder,
            Sort = 10,
            NodeAction = NavigateScreenAction.To<WidgetListScreen>().With(new WidgetsQuery())
        };
    }

    public override IEnumerable<NavigationNode> GetSubNodes(NavigationNodePath parentNodePath)
        => Enumerable.Empty<NavigationNode>();
}
```

> **`.With(query)` is not optional.** A node action of just `NavigateScreenAction.To<TScreen>()` compiles,
> deploys, and navigates — and the screen then renders its chrome (title, breadcrumb, action menu) with
> **"No results found — There are no records to display"**. There is no error anywhere; the screen simply
> has no data source. Verified by hitting it: the same node with `.With(new WidgetsQuery())` renders the
> rows immediately.

Icon names are members of the `Icon` type and are **not** free text — `Icon.List`, for instance, does not
exist. Let IntelliSense complete them rather than guessing.

To hang your section off a *built-in* area instead of your own, use that area's type as the generic
argument (`NavigationSection<ProductsArea>`).

## Extending navigation and menus you do not own

**A section on someone else's area.** Point `NavigationSection<TArea>` at their area type, then provide a
node provider for your section. Override `ShouldShow()` to hide it conditionally — on a license feature,
or on whether any relevant data exists.

**A node under someone else's node.** You do not have to own a node to add children to it: the framework
collects nodes from *all* providers targeting a section. Target their **section**, return nothing from
`GetRootNodes()`, and answer only when the parent path matches the node you want to extend.

```csharp
public sealed class ContentSettingsExtensionNodeProvider : NavigationNodeProvider<AreasSection>
{
    private const string ContentSettingsRootId = "Content_Settings";

    public override IEnumerable<NavigationNode> GetRootNodes() => [];

    public override IEnumerable<NavigationNode> GetSubNodes(NavigationNodePath parentNodePath)
    {
        if (parentNodePath is null)
            yield break;

        if (string.Equals(parentNodePath.Last, ContentSettingsRootId, StringComparison.OrdinalIgnoreCase))
        {
            yield return new NavigationNode
            {
                Name = "Pdex widgets",
                Id = "Pdex_InjectedSettingsNode",
                Icon = Icon.Sync,
                Sort = 200,          // high Sort -> after the built-in children
                NodeAction = NavigateScreenAction.To<WidgetListScreen>().With(new WidgetsQuery())
            };
        }
    }
}
```

Verified: this places the node under Dynamicweb's own **Settings → Areas → Content** node, after its
built-in children. `AreasSection` is `NavigationSection<SettingsArea>` in `Dynamicweb.Application.UI`.

> Built-in node IDs follow a `{Prefix}_{Name}` convention (`Content_Settings`, `Content_Styles`), but they
> are an implementation detail of the assembly that owns them and can change between versions. Targeting
> your own nodes is safe; targeting built-in ones is best-effort.

**An entry in someone else's Actions menu.** Use the specialised injector bases rather than the plain
`ScreenInjector<T>`:

| Override | Where it appears |
|---|---|
| `ListScreenInjector<TScreen, TRowModel>.GetScreenActions()` | the screen's toolbar / Actions menu |
| `ListScreenInjector<TScreen, TRowModel>.GetListItemActions(model)` | a row's context menu |
| `ListScreenInjector<TScreen, TRowModel>.GetCell(propertyName, model)` | a single cell's rendering |
| `EditScreenInjector<TScreen, TModel>.GetScreenActions()` | an edit screen's Actions menu |
| `EditScreenInjector<TScreen, TModel>.GetEditor(propertyName, model)` | one field's editor |

```csharp
public sealed class ApiKeyListActionInjector : ListScreenInjector<ApiKeyListScreen, ApiKeyDataModel>
{
    public override IEnumerable<ActionGroup>? GetScreenActions() => new ActionGroup[]
    {
        new()
        {
            Name = "PdexActions",       // see the warning below
            Title = "Pdex",
            Nodes =
            [
                new ActionNode
                {
                    Name = "Go to Pdex widgets",
                    Icon = Icon.Truck,
                    Sort = 500,
                    NodeAction = NavigateScreenAction.To<WidgetListScreen>().With(new WidgetsQuery())
                }
            ]
        }
    };
}
```

Verified end-to-end: the entry renders in the Actions menu of Dynamicweb's own API Keys list alongside
their "Manage columns", and clicking it navigates to the target screen.

> **Set `ActionGroup.Name` — this one bites.** Verified by A/B on a live solution: with only a `Title`,
> the injected group rendered **dimmed and clicking it did nothing**. Adding `Name = "PdexActions"` and
> redeploying made the very same entry navigate correctly. `Name` is the group's logical name and
> "affects selection-dependency on lists", so a nameless group is treated as requiring a row selection
> and stays inert until one exists. There is no error and no log entry — the item simply looks slightly
> greyed and ignores clicks.

**Action types** for any `NodeAction`:

```csharp
NavigateScreenAction.To<TScreen>().With(query)
RunCommandAction.For(new MyCommand()).WithReloadOnSuccess(ReloadType.Workspace)
ConfirmAction.For(innerAction, "Title?", "Body text")
OpenSlideOverAction.To<TScreen>().With(query).WithOnSelectAction(...)
```

For a bulk action driven by a multi-select, the command must accept the selected ids — use
`RunCommandAction.ForCommandAndProperty<TCommand>(c => c.Ids)` so the framework knows it operates on a
selection. And keep `GetListItemContextActions(model)` cheap: it runs once per row, so use what is
already on `model` rather than querying.

## Step 5 — Extending a screen you do not own

`ScreenInjector<TScreen>` hooks into a screen as it is built. Two virtual methods:

- `OnBefore(TScreen screen)` — before the content is built
- `OnAfter(TScreen screen, UiComponentBase content)` — after, to post-process the component tree

```csharp
using Dynamicweb.CoreUI;
using Dynamicweb.CoreUI.Layout;
using Dynamicweb.CoreUI.Screens;

public sealed class OrderEditScreenInjector : ScreenInjector<OrderEditScreen>
{
    public override void OnAfter(OrderEditScreen screen, UiComponentBase content)
    {
        if (content is not ScreenLayout layout)
            return;

        if (layout.Root is TabContainer tabs)
            tabs.GetOrAddTab("Acme").Section.AddGroup(/* widget or list display */);

        if (content.TryGet<Section>(out var section))
        {
            // add widgets to the existing section
        }
    }
}
```

The pattern in Dynamicweb's own injectors is defensive throughout: bail out unless the component tree is
the shape you expect, and check `screen.Model` for null. Copy that habit — an injector that throws breaks
someone else's screen.

`EditScreenInjector<TScreen, TModel>` and `ListScreenInjector<TScreen, TRowModel>` are specialised bases
for field- and cell-level work.

## Step 6 — Deploy

There is no registration step: `AddInManager` discovers areas, sections, node providers, screens, queries,
commands and injectors automatically. That convenience is also the trap — nothing tells you when
discovery failed.

```bash
dotnet build -c Release
dw install ./bin/Release/net10.0/Acme.AdminUi.dll --output json \
  --host <solution> --apiKey "$(cat ~/.dw-apikey)"
```

Omit `-q`: a queued install defers activation to the next recycle, so your screen will not appear until
then. See the dw-cli skill for the full command reference.

## Step 7 — Verify (mandatory)

**`dw install` reporting `ok: true` means the file was uploaded and the API accepted it. It is not
evidence that your code loaded, and it is never evidence that your screen works.**

Verify in this order — each step rules out a different failure:

**1. Did the assembly load?** Check `/Files/System/Log/AddInManager/TypeLoadErrors.log`. A version
mismatch appears there as `MyAddIn (Context) ReflectionTypeLoadException Could not load file or assembly
'Dynamicweb.Core, Version=...'`. The `(Context)` suffix is normal — add-ins load into their own
`AssemblyLoadContext`. No entry naming your assembly means it loaded.

**2. Did your area register?** This one *is* checkable over the API, without an admin login. The
`NavigationByPath` query resolves an area by the path segment `/{ClassNameWithoutAreaSuffix}` — so
`PdexArea` is reachable at `/Pdex`:

```bash
curl -s -H "Authorization: Bearer $KEY"   "https://<host>/Admin/Api/NavigationByPath?Path=/Pdex"
```

Tested before and after a deploy: beforehand it returns HTTP 500
`Unable to resolve area from path: /Pdex`; afterwards HTTP 200 with `title` set to your area's `Name`.
That single call proves `AddInManager` found your `AreaBase` subclass in the deployed assembly, which is
the part most likely to have failed.

> **It does not verify sections or nodes.** `sections` and `nodes` come back empty from that endpoint even
> for a core area like `/Content`, at any path depth — the navigation tree needs an authenticated admin
> user context that an API key does not carry. Do not read empty arrays as "my section is missing".

**3. Does the screen render?** Open the admin, navigate to your node, and look at it. For sections, nodes
and the screen itself this is the only real test — and it is worth doing, because the two most common
failures at this stage (an empty list, a node that leads nowhere) produce no error at all.

The whole chain in this skill has been verified this way on a live 10.29.1 solution: an area appeared in
the sidebar at its `Sort` position, its section and node rendered in the tree, and the node opened a list
screen showing the rows its query returned, with the columns from `GetViewMappings()`.

**4. Did anything throw?** Errors during rendering land in the `GeneralLog` table. Query it with
`LogEventByFilters` — see the debugging section of the dw-extend skill for the exact filters.

If the node is missing but the assembly loaded, suspect the routing decision from Step 1 — a screen
subclassing a core screen, or a node provider whose section type does not match a real area.

## Common mistakes

| Symptom | Cause |
|---|---|
| Screen deploys but never renders | Subclassed a core screen instead of using an injector |
| `CS0311` on your `NavigationSection<>` | Missing `using Dynamicweb.CoreUI.Application;` so `AreaBase` did not resolve |
| `CS0117: 'Icon' does not contain a definition for ...` | Invented an icon name |
| Nothing appears, no error anywhere | Built against newer packages than the solution runs — check `TypeLoadErrors.log` |
| Screen opens but says "No results found" | Node action has no query bound — add `.With(new YourQuery())` |
| Change appears only after a recycle | Installed with `-q` |
| Someone else's screen breaks after your deploy | Injector threw; add the shape and null checks |

## Reference

- [Custom UI landing](https://doc.dynamicweb.dev/documentation/extending/administration-ui/index.html)
- [Screen architecture](https://doc.dynamicweb.dev/documentation/extending/administration-ui/screenconcept.html)
- [Creating custom screens](https://doc.dynamicweb.dev/documentation/extending/administration-ui/screens.html)
- [Screen injectors](https://doc.dynamicweb.dev/documentation/extending/administration-ui/screen-injectors.html)
- [Area list](https://doc.dynamicweb.dev/documentation/extending/administration-ui/arealist.html) ·
  [Area tree](https://doc.dynamicweb.dev/documentation/extending/administration-ui/areatree.html) ·
  [Action menu](https://doc.dynamicweb.dev/documentation/extending/administration-ui/action-menu.html)

Reading real implementations beats guessing: `Dynamicweb.Application.UI` in the platform source holds 123
screens, 139 queries and 103 commands. `LicenseFeatureListScreen` and `OAuthClientDeleteCommand` are good
minimal examples.
