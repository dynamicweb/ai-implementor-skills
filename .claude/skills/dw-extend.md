---
name: dw-extend
description: Guide for building best-practice extensions for Dynamicweb and Dynamicweb Swift — covers NotificationSubscribers, Providers, ScheduledTasks, Razor template overrides, CSS customization, and ViewModel extensions.
---

You are a Dynamicweb extension specialist. When the user asks you to build, scaffold, or advise on extending Dynamicweb or Dynamicweb Swift, follow this skill precisely.

## Your role

Help the developer produce a clean, upgrade-safe, best-practice implementation. Always:
- Ask clarifying questions if the intent is unclear before generating code
- Choose the **least invasive** approach that satisfies the requirement
- Prefer **upgradeability** over convenience — never advise editing Swift core files directly
- Use C# for all backend extension code
- Use Razor (`.cshtml`) for frontend template overrides

---

## Step 1 — Classify the request

Determine which extension type(s) are needed. A request may require more than one.

| What the developer wants | Extension type |
|---|---|
| React to a platform event (order placed, user login, page render, etc.) | NotificationSubscriber |
| Replace or override a core calculation (price, tax, shipping, stock, feed) | Provider |
| Run logic on a schedule or via the admin task runner | ScheduledTask AddIn |
| Change or add frontend presentation without touching Swift source | Swift CSS or Razor override |
| Add custom data to a Razor template context | ViewModel extension |
| Add custom UI, screens, or buttons in the Dynamicweb admin | Administration UI extension |
| Create custom database tables or migrate schema | UpdateProvider + Database query |

---

## Step 2 — Backend extensions (C# class library)

### Project setup

Extensions live in a **separate C# class library** that references Dynamicweb NuGet packages. Never put extension code inside the Dynamicweb core installation.

```xml
<!-- .csproj -->
<ItemGroup>
  <PackageReference Include="Dynamicweb" Version="10.*" />
  <!-- Add Dynamicweb.Ecommerce, Dynamicweb.Content etc. as needed -->
</ItemGroup>
```

---

### NotificationSubscriber

Use when you need to hook into a platform event and run code alongside (not instead of) the normal flow.

**Base class:** `Dynamicweb.Extensibility.Notifications.NotificationSubscriber`
**Attribute:** `[Subscribe(NotificationName)]`
**Override:** `OnNotify(string notification, NotificationArgs args)`

```csharp
using Dynamicweb.Extensibility.Notifications;

[Subscribe(Dynamicweb.Ecommerce.Notifications.Ecommerce.Order.Placed)]
public class OrderPlacedSubscriber : NotificationSubscriber
{
    public override void OnNotify(string notification, NotificationArgs args)
    {
        if (args is not Dynamicweb.Ecommerce.Notifications.Ecommerce.Order.PlacedArgs placedArgs)
            return;

        var order = placedArgs.Order;
        // your logic here
    }
}
```

**Key rules:**
- One class can subscribe to multiple notifications — add multiple `[Subscribe]` attributes
- Cast `args` to the specific args type for that notification before use; bail out safely if cast fails
- Do not perform heavy I/O synchronously in `OnNotify` unless the notification explicitly supports it — offload to a background service if needed
- Subscribers are auto-discovered — no registration required

**Finding notification names:**
Browse `Dynamicweb.Ecommerce.Notifications`, `Dynamicweb.Content.Notifications`, `Dynamicweb.Security.Notifications` etc. in the API reference at https://doc.dynamicweb.dev/api/

---

### Provider

Use when you want to **replace** a core subsystem (price calculation, shipping, tax, stock levels, payment, etc.) with your own implementation. DynamicWeb will discover the provider and make it selectable in the admin.

**Pattern:** inherit the relevant base class, decorate with `[AddInName]` and `[AddInLabel]`.

```csharp
using Dynamicweb.Extensibility.AddIns;
using Dynamicweb.Ecommerce.Prices;

[AddInName("MyCompany.CustomPriceProvider")]
[AddInLabel("My Custom Price Provider")]
public class CustomPriceProvider : PriceProvider
{
    public override PriceRaw FindPrice(PriceContext context)
    {
        // return null to fall through to next provider
        // return a PriceRaw to use this price
        return null;
    }
}
```

**Common provider base classes:**

| Use case | Base class | Key method |
|---|---|---|
| Custom price logic | `Dynamicweb.Ecommerce.Prices.PriceProvider` | `FindPrice()` |
| Custom shipping | `Dynamicweb.Ecommerce.Orders.ShippingProvider` | varies |
| Custom tax | `Dynamicweb.Ecommerce.Prices.TaxProvider` | varies |
| Custom stock | `Dynamicweb.Ecommerce.Stocks.StockLevelProvider` | varies |
| Custom cart calculation | `Dynamicweb.Ecommerce.Prices.CartCalculationProvider` | varies |
| Payment handler | `Dynamicweb.Ecommerce.CheckoutHandlers.CheckoutHandler` | varies |
| Custom product feed | `Dynamicweb.Ecommerce.Feeds.FeedProvider` | varies |
| Custom logging | `Dynamicweb.Logging.ILoggerProvider` (interface) | varies |
| Custom navigation | `Dynamicweb.Content.Navigation.NavigationProvider` | varies |
| Custom index | `Dynamicweb.Indexing.IndexProviderBase` | varies |

**Key rules:**
- Providers are auto-discovered but must be **selected in the admin UI** before they activate
- Add `[AddInDescription("...")]` for admin documentation
- Use `[AddInConfigurable]` properties (with `AddInParameter` attribute) to expose configuration in the admin

---

### ScheduledTask AddIn

Use when you need logic to run on a timer or to appear in the Dynamicweb Scheduled Tasks admin.

**Base class:** `Dynamicweb.Extensibility.AddIns.BaseScheduledTaskAddIn`
**Attributes:** `[AddInName]`, `[AddInLabel]`, `[AddInDescription]`
**Method:** override `Run()` — return `true` on success, `false` on failure

```csharp
using Dynamicweb.Extensibility.AddIns;

[AddInName("MyCompany.DataSyncTask")]
[AddInLabel("Data Sync Task")]
[AddInDescription("Syncs product data from the external ERP system.")]
public class DataSyncTask : BaseScheduledTaskAddIn
{
    [AddInParameter("Batch size")]
    [AddInParameterEditor(typeof(IntegerNumberParameterEditor), "")]
    public int BatchSize { get; set; } = 100;

    public override bool Run()
    {
        try
        {
            // your logic here
            Dynamicweb.Logging.LogManager.System.GetLogger(typeof(DataSyncTask))
                .Info($"DataSyncTask completed. BatchSize={BatchSize}");
            return true;
        }
        catch (Exception ex)
        {
            Dynamicweb.Logging.LogManager.System.GetLogger(typeof(DataSyncTask))
                .Error("DataSyncTask failed", ex);
            return false;
        }
    }
}
```

**Key rules:**
- `Run()` is called synchronously — wrap heavy work in try/catch and log exceptions
- Expose configuration via `[AddInParameter]` + `[AddInParameterEditor]` properties
- The task appears in admin after the DLL is deployed — the admin user creates a schedule for it
- Return `false` (not throw) to signal failure — this allows the scheduler to record it properly

---

### Services — accessing platform data

When you need data that is not available on a ViewModel or in a `NotificationArgs` object, use the appropriate static `Services` class. Services are the correct way to read and write platform data from any extension type (NotificationSubscriber, Provider, ScheduledTask, etc.).

**Pattern:** call the static service accessor, then call methods on the returned service instance.

```csharp
// Content
using Dynamicweb.Content;

var page = Services.Pages.GetPage(pageId);
var area = Services.Areas.GetArea(areaId);
var paragraphs = Services.Paragraphs.GetParagraphsByPageId(pageId);
var items = Services.Items.GetItemsByItemType("MyItemType");

// Ecommerce
using Dynamicweb.Ecommerce;

var product = Services.Products.GetProduct(productId, variantId, languageId);
var order = Services.Orders.GetOrder(orderId);
var orders = Services.Orders.GetOrdersByCustomerId(customerId);
var cart = Services.Carts.GetCart(cartId);
var currency = Services.Currencies.GetCurrency(currencyCode);
var discount = Services.Discounts.GetDiscount(discountId);

// Users
using Dynamicweb.Security.UserManagement;

var user = UserManagementServices.Users.GetUserById(userId);
var group = UserManagementServices.UserGroups.GetGroupById(groupId);
var addresses = UserManagementServices.UserAddresses.GetUserAddresses(userId);

// Forms
using Dynamicweb.Forms;

var form = FormsServices.Forms.GetFormById(formId);
var submits = FormsServices.Submits.GetSubmitsByFormId(formId);

// Email Marketing
using Dynamicweb.EmailMarketing;

var flow = Services.Flows.GetFlowById(flowId);
var recipients = Services.FlowRecipients.GetRecipientsByFlowId(flowId);
```

**Available service entry points:**

| Namespace | Entry point | Key services |
|---|---|---|
| `Dynamicweb.Content` | `Services.*` | `Pages`, `Areas`, `Paragraphs`, `Items`, `Grids`, `Colors`, `Comments` |
| `Dynamicweb.Ecommerce` | `Services.*` | `Products`, `Orders`, `OrderLines`, `Carts`, `Prices`, `Discounts`, `Vouchers`, `Variants`, `Countries`, `Currencies`, `Shippings`, `Taxes`, `Payments`, `StockService`, `Manufacturers`, `Loyalty` + 70 more |
| `Dynamicweb.Security.UserManagement` | `UserManagementServices.*` | `Users`, `UserGroups`, `UserAddresses`, `UserImpersonations`, `UserAndGroupTypes` |
| `Dynamicweb.Forms` | `FormsServices.*` | `Forms`, `Fields`, `Options`, `Submits`, `SubmitData` |
| `Dynamicweb.EmailMarketing` | `Services.*` | `Flows`, `FlowFolders`, `FlowSteps`, `FlowRecipients`, `FlowStepHistory` |

**Key rules:**
- Services are singletons managed by the framework — never instantiate them directly with `new`
- Services handle caching internally — do not cache service results aggressively yourself unless you have a measured performance need
- For writes (Save, Delete), always check whether the service method accepts the full entity or just an ID, and pass back the entity you retrieved to avoid overwriting fields you didn't intend to change
- Prefer services over direct `Database.ExecuteReader` calls for data that services already expose — services apply business logic, permission checks, and caching that raw SQL bypasses

**API reference:** https://doc.dynamicweb.dev/api/

---

### ViewModel Extension

Use when a Razor template needs data that is not in the standard ViewModel.

**Option A — Extension method** (for computed/derived values, no serialization needed):

```csharp
public static class ProductViewModelExtensions
{
    public static string FormattedStockLabel(this ProductViewModel model)
        => model.StockLevel > 0 ? $"In stock ({model.StockLevel})" : "Out of stock";
}
```

Usage in template: `@Model.FormattedStockLabel()`

**Option B — Subclass** (for extra properties, JSON serialization, constructor injection):

```csharp
[AddInName("MyProductViewModel")] // use ItemType SystemName if item-specific
public class MyProductViewModel : ProductViewModel
{
    public string ExtraField => "computed value";
}
```

**Key rule:** You cannot override values that the base ViewModel sets during instantiation — create wrapper/new properties instead.

---

## Step 3 — Database extensions

### UpdateProvider — schema migrations

Use when your extension needs its own database tables or columns. The `UpdateProvider` runs automatically on application startup and applies any pending updates idempotently — safe to deploy multiple times.

**Base class:** `Dynamicweb.Updates.UpdateProvider`
**Namespace:** `Dynamicweb.Updates`
**Override:** `GetUpdates()` — return all schema changes this extension owns

```csharp
using Dynamicweb.Updates;

public sealed class MyExtensionUpdateProvider : UpdateProvider
{
    public override IEnumerable<Update> GetUpdates() => new List<Update>
    {
        // Create a custom table (only runs if table does not exist)
        SqlUpdate.AddTable(
            id: "A1B2C3D4-E5F6-7890-ABCD-EF1234567890",
            provider: this,
            tableName: "MyExtension_Product",
            tableDefinition: """
                MyExtension_ProductId INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
                MyExtension_ProductExternalId NVARCHAR(255) NOT NULL,
                MyExtension_ProductCreated DATETIME NOT NULL DEFAULT GETDATE()
                """
        ),

        // Add a column (only runs if column does not exist)
        SqlUpdate.AddColumn(
            id: "B2C3D4E5-F6A7-8901-BCDE-F12345678901",
            provider: this,
            tableName: "MyExtension_Product",
            columnName: "MyExtension_ProductSyncedAt",
            columnDefinition: "DATETIME NULL"
        ),

        // Change a column definition (only runs if column exists)
        SqlUpdate.ChangeColumn(
            id: "C3D4E5F6-A7B8-9012-CDEF-123456789012",
            provider: this,
            tableName: "MyExtension_Product",
            columnName: "MyExtension_ProductExternalId",
            columnDefinition: "NVARCHAR(500) NOT NULL"
        ),

        // Add an index
        SqlUpdate.AddIndex(
            id: "D4E5F6A7-B8C9-0123-DEFA-234567890123",
            provider: this,
            tableName: "MyExtension_Product",
            indexName: "IX_MyExtension_Product_ExternalId",
            indexDefinition: "MyExtension_ProductExternalId ASC"
        ),

        // Raw SQL update with an optional conditional (skip if condition returns rows)
        new SqlUpdate(
            id: "E5F6A7B8-C9D0-1234-EFAB-345678901234",
            provider: this,
            sql: "UPDATE MyExtension_Product SET MyExtension_ProductSyncedAt = GETDATE() WHERE MyExtension_ProductSyncedAt IS NULL",
            sqlConditional: "SELECT 1 WHERE NOT EXISTS (SELECT 1 FROM MyExtension_Product WHERE MyExtension_ProductSyncedAt IS NULL)"
        ),
    };
}
```

**Key rules:**
- Every update `id` must be a **unique GUID across all UpdateProviders** in the entire solution — generate a fresh one per update, never reuse
- UpdateProviders are **auto-discovered** — no registration needed
- Operations are **idempotent** — `AddTable`, `AddColumn`, `AddIndex` all check existence before acting
- `DropColumn` is irreversible — use with care and only when the column is confirmed unused
- Prefix table and column names with your extension name to avoid collisions with Dynamicweb core tables
- Never modify Dynamicweb core tables via an UpdateProvider — only manage tables your extension owns

**Other update types:**

| Type | Use case |
|---|---|
| `SqlUpdate` | DDL and DML schema changes |
| `SettingUpdate` | Insert or change settings in the Dynamicweb settings store |
| `MethodUpdate` | Run arbitrary C# code as a one-time migration step |
| `FileUpdate` | Copy or modify files during deployment |

---

### Querying the database

Use `Dynamicweb.Database` (static class) for raw SQL access to both Dynamicweb core tables and your custom tables.

```csharp
using Dynamicweb;
using System.Data;

// SELECT — returns an IDataReader
using var reader = Database.ExecuteReader("SELECT MyExtension_ProductId, MyExtension_ProductExternalId FROM MyExtension_Product WHERE MyExtension_ProductSyncedAt IS NULL");
while (reader.Read())
{
    var id = reader.GetInt32(0);
    var externalId = reader.GetString(1);
}

// Scalar — single value
var count = Database.ExecuteScalar("SELECT COUNT(*) FROM MyExtension_Product");

// INSERT / UPDATE / DELETE — returns rows affected
Database.ExecuteNonQuery($"UPDATE MyExtension_Product SET MyExtension_ProductSyncedAt = GETDATE() WHERE MyExtension_ProductId = {id}");
```

**Key rules:**
- Always use parameterized queries or sanitize inputs when values come from user input — raw string interpolation is only safe for internal/trusted values
- Prefer `using` blocks around `IDataReader` to ensure the connection is released
- `Database.ExecuteNonQuery` returns the number of rows affected — check it when correctness matters
- For complex read scenarios, prefer mapping results to a typed list immediately after the reader loop rather than passing the open reader around

---

## Step 4 — Dynamicweb Swift frontend extensions

### File structure reference

Understanding where files live is essential before touching anything.

```
Files/
├── GlobalSettings.config              # Main DW config (DB connection, site settings)
├── GlobalSettings.Ecom.config         # Ecommerce-specific global settings
├── Images/
│   └── Icons/
│       ├── *.svg                      # UI icon set used across Swift
│       ├── Flags/                     # Country/language flag SVGs
│       └── LoginProviders/            # Social/external login icons
├── System/
│   ├── Items/                         # Item type XML field schema definitions
│   ├── Repositories/                  # Search index configurations
│   │   ├── Content/                   # Page/content index
│   │   ├── Files/                     # File asset index
│   │   ├── Post/                      # Blog/post index
│   │   ├── ProductsBackend/           # Product index for admin
│   │   ├── ProductsFrontend/          # Product index for frontend
│   │   └── Secondary users/           # Secondary user account index
│   └── Styles/
│       └── ColorSchemes/              # Color scheme definitions → generates --dw-color-* CSS variables
└── Templates/
    ├── CookieWarning/                 # Cookie consent banner template
    ├── Items/                         # Item type editor templates (admin UI)
    └── Designs/
        └── Swift-v2/                  # ← entire Swift v2 design lives here
            ├── Assets/
            │   ├── css/               # Swift.css + custom.css (your overrides go here)
            │   ├── js/                # Swift JavaScript
            │   ├── images/            # Email assets, file type icons, item type previews
            │   └── lib/               # Bundled JS libraries:
            │       ├── alpinejs/      #   Alpine.js (reactivity)
            │       ├── bootstrap/     #   Bootstrap (grid + components)
            │       ├── flatpickr/     #   Date picker
            │       ├── htmx.org/      #   htmx (AJAX)
            │       └── swiffy-slider/ #   Carousel/slider
            ├── Components/            # Shared Razor partials reused across item types
            ├── Custom/                # ← YOUR customization layer (project-specific files)
            ├── Forms/                 # Dynamicweb Forms rendering templates
            ├── Grid/
            │   ├── Page/              # Page layout — row definitions + row templates
            │   └── Email/             # Email layout — row definitions + row templates
            ├── Navigation/            # Vertical nav + breadcrumb templates
            ├── Paragraph/             # One folder per Swift item type (content building blocks):
            │   ├── Swift-v2_Accordion/
            │   ├── Swift-v2_Button/
            │   ├── Swift-v2_Card/
            │   ├── Swift-v2_Image/
            │   ├── Swift-v2_Navigation/
            │   ├── Swift-v2_Poster/
            │   ├── Swift-v2_Slider/
            │   ├── Swift-v2_Text/
            │   ├── Swift-v2_TextAndImage/
            │   └── ... (30+ item types total)
            ├── QueryPublisher/        # Post/blog list query templates
            ├── Swift-v2_Email/        # Transactional + marketing email templates
            ├── UserManagement/        # Login, create profile, view profile pages
            ├── Users/                 # Full user self-service:
            │   ├── UserAddresses/     #   Address list + edit
            │   ├── UserAuthentication/#   Login flow
            │   ├── UserChangePassword/#   Password change
            │   ├── UserCreate/        #   Registration + confirmation email
            │   ├── UserEdit/          #   Profile editing
            │   ├── UserForgotPassword/#   Forgot/reset password flow
            │   ├── UserGroups/        #   User group management
            │   └── UserView/          #   User list + detail views
            ├── eCom/                  # Ecommerce templates:
            │   ├── CustomerCenter/    #   B2B customer center
            │   ├── CustomerExperienceCenter/ # Orders, favorites, saved cards
            │   ├── IntegrationCustomerCenter/
            │   └── ProductCatalog/    #   Product list + slider
            └── eCom7/
                ├── CartV2/            # Checkout flow (cart steps + order confirmation emails)
                └── ShippingProvider/  # Shipping method selection template
```

**Complete item type reference (by category):**

| SystemName | Display name | Notes |
|---|---|---|
| **Swift-v2 (page level)** |||
| `Swift-v2_Master` | Master | Master layout page |
| `Swift-v2_Page` | Page | Standard content page |
| `Swift-v2_PageProperties` | Page properties | SEO/meta settings for a page |
| `Swift-v2_ServicePage` | Service page | Technical/utility page |
| **Swift-v2/Layout** |||
| `Swift-v2_Header` | Header | Site header layout |
| `Swift-v2_Footer` | Footer | Site footer layout |
| **Swift-v2/Grid** |||
| `Swift-v2_Row` | Row with columns | Fixed column grid row |
| `Swift-v2_RowFlex` | Row with flexible columns | Flexible column grid row |
| **Swift-v2/Paragraphs/Standard** |||
| `Swift-v2_Accordion` | Accordion | Collapsible FAQ-style sections |
| `Swift-v2_Accordion_Item` | Accordion Item | Child item for Accordion |
| `Swift-v2_App` | App | Wrapper for Dynamicweb apps |
| `Swift-v2_Blockquote` | Blockquote | Styled quote block |
| `Swift-v2_Button` | Button | CTA button |
| `Swift-v2_Card` | Card | Image + text card |
| `Swift-v2_CookieNotice` | Cookie notice | Cookie consent banner |
| `Swift-v2_Employee` | Employee | Employee profile card |
| `Swift-v2_Feature` | Feature | Icon + text feature highlight |
| `Swift-v2_HelloUser` | Hello user | Personalised greeting |
| `Swift-v2_Image` | Image | Standalone image block |
| `Swift-v2_ImpersonationBar` | Impersonation bar | B2B user impersonation indicator |
| `Swift-v2_Information_Item` | Information Item | Child list item |
| `Swift-v2_LocationsMap` | Locations map | Multi-location map |
| `Swift-v2_Logo` | Logo | Site logo |
| `Swift-v2_PostList` | Post list | Blog/news post list |
| `Swift-v2_Poster` | Poster | Full-width hero/poster block |
| `Swift-v2_ProductComponentSlider` | Product component slider | Slider of product components |
| `Swift-v2_ProductGroupGrid` | Products group grid | Ecom product group grid |
| `Swift-v2_ProductGroupSlider` | Products group slider | Ecom product group slider |
| `Swift-v2_SearchField` | Search field | Site search input |
| `Swift-v2_SimpleMap` | Simple map | Single-location map (no API key needed) |
| `Swift-v2_Slider` | Slider | Swiffy Slider carousel |
| `Swift-v2_Slider_Item` | Slider Item | Child item for Slider |
| `Swift-v2_Text` | Text | Rich text block |
| `Swift-v2_TextAndImage` | Text and image | Side-by-side text + image |
| `Swift-v2_VideoPlayer` | Video player | Embedded video, no layout options |
| `Swift-v2_VideoPoster` | Video poster | Video with poster image |
| **Swift-v2/Paragraphs/Navigation** |||
| `Swift-v2_BreadcrumbNavigation` | Breadcrumb navigation | Page breadcrumb trail |
| `Swift-v2_Favorites` | Favorites | Saved favorites list |
| `Swift-v2_MenuProductGroupImages` | Menu: Ecom with Product Group Images | Mega menu with group images (2 levels) |
| `Swift-v2_MenuRelatedContent` | Menu: Ecom with Related Content | Mega menu with related content (3 levels) |
| `Swift-v2_MiniCart` | Mini cart | Cart icon + item count |
| `Swift-v2_MyAccount` | My account | User account dropdown/link |
| `Swift-v2_Navigation` | Navigation | Main site navigation |
| `Swift-v2_OffCanvasNavigation` | Off-Canvas navigation | Mobile slide-out navigation |
| `Swift-v2_Preferences` | Preferences | Language/currency switcher |
| `Swift-v2_SignIn` | Sign in | Login link/button |
| **Swift-v2/Paragraphs/ProductDetails** |||
| `Swift-v2_ProductMedia` | Media | Product images with thumbnails |
| `Swift-v2_ProductMediaGallery` | Media gallery | Full product image gallery |
| `Swift-v2_ProductMediaTable` | Media table | Product media in table layout |
| `Swift-v2_ProductFieldDisplayGroupsAccordion` | Product field display groups accordion | Product specs as accordion |
| `Swift-v2_RelatedProductsList` | Related products list view | Related/cross-sell product list |
| **Swift-v2/Paragraphs/ProductList** |||
| `Swift-v2_ProductListFacets` | Facets | Filter facets panel |
| `Swift-v2_ProductListSelectedFacets` | Facets (selected) | Active filter chips |
| `Swift-v2_ProductListInfo` | Group description (info) | Ecom group title + description |
| `Swift-v2_ProductListHeader` | Group header | Group name + product count |
| `Swift-v2_ProductListGroupImage` | Group image | Ecom group image |
| `Swift-v2_ProductGroupList` | Group navigation | Product group links |
| `Swift-v2_ProductListGroupPoster` | Group poster | Large group image poster |
| `Swift-v2_ProductListNavigation` | Navigation | Pagination / list navigation |
| `Swift-v2_ProductListItemRepeater` | Product component repeater | Custom list view — repeats a component |
| `Swift-v2_ProductListSortBy` | Sort by | Sort order selector |
| **Swift-v2/Paragraphs/ProductPartial** |||
| `Swift-v2_ProductAddToCart` | Add to cart | Add to cart button |
| `Swift-v2_ProductAddToDownloadCart` | Add to download cart | Add to download cart |
| `Swift-v2_ProductAddToFavorites` | Add to favorites | Wishlist/favorites button |
| `Swift-v2_ProductAddToQuoteCart` | Add to quote cart | Quote cart button |
| `Swift-v2_BackInStockNotification` | Back in stock notification | Out-of-stock alert sign-up |
| `Swift-v2_ProductBom` | BOM | Bill of materials |
| `Swift-v2_ProductDefaultImage` | Default product image | Fallback product image |
| `Swift-v2_ProductDownloadData` | Download data form | Downloadable data form |
| `Swift-v2_ProductDownloadPublication` | Download publication | Downloadable publication |
| `Swift-v2_ProductFieldDisplayGroups` | Product field display groups | Product specs as list |
| `Swift-v2_ProductHeader` | Name | Product name/title |
| `Swift-v2_ProductLongDescription` | Description (long) | Full product description (detail page only) |
| `Swift-v2_ProductShortDescription` | Description (short) | Product teaser text |
| `Swift-v2_ProductNumber` | Number | Product number/SKU |
| `Swift-v2_ProductPrice` | Price | Default product price |
| `Swift-v2_ProductPriceTable` | Price table | Quantity price matrix |
| `Swift-v2_ProductQuantity` | Quantity | Static quantity (for BOM items) |
| `Swift-v2_ProductStaticVariants` | Static variant view | Non-interactive variant display |
| `Swift-v2_ProductStock` | Stock | Stock level indicator |
| `Swift-v2_ProductStockLocations` | Stock locations | Per-location stock levels |
| `Swift-v2_ProductVariantSelector` | Selectable variant options | Variant selector (blocks add-to-cart until complete; detail page only) |
| `Swift-v2_VariantGroups` | Variant Groups | Variant group display |
| **Swift-v2/Ecom** |||
| `Swift-v2_Shop` | Shop | Product list page with Product Catalog App |
| `Swift-v2_ProductList` | Product list | Product listing configuration |
| `Swift-v2_ProductDetails` | Product details | Product detail page configuration |
| `Swift-v2_CartSummary` | Cart Summary | Cart totals summary |
| `Swift-v2_ProductComponent` | Product component | Reusable component for detail + list cards |
| `Swift-v2_ProductComponentSelector` | Product component selector | Selects which component to render |
| `Swift-v2_ProductListComponent` | Product list component | Reusable component for list items |
| `Swift-v2_ProductListComponentSelector` | Product list component selector | Selects which list component to render |
| **Swift-v2/Utilities** |||
| `Swift-v2_Cart` | Cart | Shopping cart page |
| `Swift-v2_CartApp` | Cart App | Cart as a Dynamicweb app |
| `Swift-v2_Checkout` | Checkout | Checkout page |
| `Swift-v2_CheckoutApp` | Checkout App | Checkout as a Dynamicweb app |
| `Swift-v2_DownloadCartApp` | Download cart App | Download cart app setup |
| `Swift-v2_ExpressBuy` | Express Buy | Quick-buy flow |
| `Swift-v2_ExpressBuyApp` | Express Buy App | Express buy as a Dynamicweb app |
| `Swift-v2_QuoteCheckoutApp` | Quote checkout App | Quote checkout as a Dynamicweb app |
| `Swift-v2_SearchPage` | Search page | Site search results page |
| `Swift-v2_DisabledDeliveryDate_Item` | Disabled Delivery Date Item | Blackout date for delivery calendar |
| **Swift-v2/Emails** |||
| `Swift-v2_Email` | Email | Email layout root |
| `Swift-v2_EmailHeader` | Email header | Email header block |
| `Swift-v2_EmailFooter` | Email footer | Email footer block |
| `Swift-v2_EmailRow` | Email Row | Email grid row |
| `Swift-v2_EmailArticle` | Article | Text + image + button block |
| `Swift-v2_EmailButton` | Button | Email CTA button |
| `Swift-v2_EmailHeading` | Heading | Email heading text |
| `Swift-v2_EmailIcons` | Icons | Icon row in email |
| `Swift-v2_EmailIcon_Item` | Icon item | Child item for Icons |
| `Swift-v2_EmailImage` | Image | Email image block |
| `Swift-v2_EmailMenu` | Menu | Email navigation menu |
| `Swift-v2_EmailMenu_Item` | Menu item | Child item for Menu |
| `Swift-v2_EmailOrderlines` | Orderlines | Order line items block |
| `Swift-v2_EmailProductCatalog` | Product catalog | Product list in email |
| `Swift-v2_EmailBackInStockNotificationProductCatalog` | Back in stock notification product catalog | Back-in-stock product block |
| `Swift-v2_EmailSpacer` | Spacer | Vertical spacing block |
| `Swift-v2_EmailText` | Text | Email text block |
| `Swift-v2_EmailUnsubscribeLink` | Unsubscribe link | Unsubscribe footer link |
| `Swift-v2_EmailViewInBrowser` | View in browser | View-in-browser link |
| **Swift-v2/Misc** |||
| `Swift-v2_ServicesFolder` | Service folder | Container for service pages |

---

**Customization entry points — in order of preference:**

| What you want to change | Where to put it |
|---|---|
| CSS variable overrides, brand colors, spacing | `Assets/css/custom.css` |
| Override an existing Razor template | Mirror the path in your own Design folder (not Swift-v2 directly) |
| Add a new item type | `Paragraph/YourItemType/` + register in `System/Items/` |
| Project-specific partials and helpers | `Custom/` |
| Color scheme (generates CSS variables) | `System/Styles/ColorSchemes/` |

---

### Ground rules

- **Never edit files in the Swift repository directly** if you intend to receive Swift updates
- All overrides go in a **separate customization layer** tracked in your own Git repository
- Prefer CSS variable overrides → then CSS selector overrides → then Razor template overrides → then new ItemTypes (increasing maintenance cost)

### Git workflow

```bash
# Add Swift as upstream so you can pull future updates
git remote add upstream https://github.com/dynamicweb/Swift.git
git fetch upstream

# Create a feature branch for your customization
git checkout -b feature/my-customization

# To pull Swift updates later
git fetch upstream
git merge upstream/main --allow-unrelated-histories
```

---

### CSS customization (`custom.css`)

**Load order:** Swift.css → Dynamicweb-generated color/typography files → **your custom.css** (wins)

**File location:** place your overrides in `custom.css` at the root of your Swift customization project. It is loaded last and takes precedence.

```css
/* Use CSS custom properties (variables) — never hard-code values */
:root {
    --dw-font-body: "My Brand Font", sans-serif;
    --dw-btn-border-radius: 0.25rem;
}

/* Scope overrides with Swift's semantic attributes for precision */
[data-dw-colorscheme="light"] [data-dw-itemtype="swift-v2_text"] {
    background-color: var(--dw-color-accent);
}

/* Target layout sections and columns */
[data-swift-gridrow] { gap: 2rem; }
[data-swift-gridcolumn] { padding: 1rem; }
```

**Available CSS variable groups:**

| Prefix | Controls |
|---|---|
| `--dw-font-*` | Font family, weight, line-height, spacing |
| `--dw-btn-*` | Button padding, radius, border width |
| `--dw-color-*` | Color scheme colors (auto-generated from backend) |

**Key rules:**
- Consume `--dw-color-*` variables instead of hex values so color scheme changes propagate
- Define brand colors as **Custom Colors** in the Dynamicweb backend — they become CSS variables automatically
- Never edit `Swift.css` or the Dynamicweb-generated CSS files

---

### Razor template overrides

Override a Swift Razor template by placing a file at the **same relative path** in your customization project. Swift resolves templates with your version taking priority.

```
/Files/Templates/Designs/Swift/   ← Swift source (do not edit)
/Files/Templates/Designs/MyCustom/ ← your overrides (same relative paths)
```

**Impact levels:**
- CSS changes — lowest risk, easiest to merge on Swift update
- Razor overrides — medium risk; when Swift updates the same template you must merge manually
- New ItemTypes — highest flexibility, highest maintenance burden

**Best practice:** override as little as possible. If you only need a small change inside a template, consider using a CSS selector or a JavaScript hook before reaching for a full Razor override.

---

### Custom ItemTypes — creating new content building blocks

When the built-in Swift item types don't cover a requirement, create a custom ItemType. Each ItemType consists of:

1. An **XML schema file** in `Files/System/Items/` — defines fields, admin UI layout, and activation rules
2. A **Razor template** in `Files/Templates/Designs/Swift-v2/Paragraph/YourItemType/` — renders the block on the frontend

#### XML schema skeleton

```xml
<?xml version="1.0" encoding="utf-16" standalone="yes"?>
<items>
  <item
    category="Swift-v2/Paragraphs/Standard"
    name="My Block"
    systemName="MyPrefix_MyBlock"
    description=""
    icon="Columns"
    iconColor="None"
    image=""
    includeInUrlIndex="False"
    pageDefaultView=""
    paragraphDefaultModule=""
    fieldForTitle="Title"
    title=""
    inherits="">

    <fields>
      <!-- field definitions -->
    </fields>

    <rules>
      <!-- activation and restriction rules -->
    </rules>

    <layout>
      <groups>
        <!-- admin UI field groupings -->
      </groups>
    </layout>
  </item>
</items>
```

**Key `<item>` attributes:**
- `systemName` — `MyPrefix_BlockName`; prefix with your project/company to avoid collisions with Swift core
- `category` — `Swift-v2/Paragraphs/Standard` for general blocks, or any custom path you define
- `fieldForTitle` — `systemName` of the field shown as the block's display name in the admin tree
- `icon` — Bootstrap Icons name (e.g. `AlignLeft`, `AspectRatio`, `Book`, `Columns`, `Image`) — see https://icons.getbootstrap.com/

#### Field definition structure

```xml
<field
  name="Display Name"
  systemName="CamelCase"
  description=""
  type="System.String, System.Private.CoreLib"
  excludeFromSearch="False"
  defaultValueCulture="en-DK"
  defaultValue="Optional default value">
  <editor type="Dynamicweb.Content.Items.Editors.EditorName, Dynamicweb">
    <editorConfuguration>
      <Parameters addin="Dynamicweb.Content.Items.Editors.EditorName">
        <Parameter addin="Dynamicweb.Content.Items.Editors.EditorName" name="ParamName" value="ParamValue" />
      </Parameters>
    </editorConfuguration>
  </editor>
</field>
```

> **Note:** `editorConfuguration` is the correct (intentional typo) spelling used by Dynamicweb — do not correct it.

**Field `type` values by data type:**

| Data type | `type` attribute |
|---|---|
| Text (most fields) | `System.String, System.Private.CoreLib` |
| Boolean (checkbox) | `System.Boolean, System.Private.CoreLib` |
| Integer | `System.Int32, System.Private.CoreLib` |
| DateTime | `System.DateTime, System.Private.CoreLib` |

#### Editor types — full reference

All editors live in `Dynamicweb.Content.Items.Editors.*`. The full editor `type` is `Dynamicweb.Content.Items.Editors.EditorName, Dynamicweb`.

| Editor | Field `type` | Key `<Parameter>` names | Purpose |
|---|---|---|---|
| `RichTextEditor` | String | `DefaultFontStyle`, `ShowToggle`, `ShowTagSelector`, `ShowStyleSelector` | Multi-line rich text with full toolbar |
| `RichTextEditorLight` | String | `DefaultFontStyle`, `DefaultTag`, `ShowToggle`, `ShowTagSelector`, `ShowStyleSelector` | Single-line heading/subtitle with limited toolbar |
| `TextEditor` | String | `ShowToggle` | Plain single-line text |
| `LongTextEditor` | String | `ShowToggle` | Plain multi-line textarea |
| `ButtonEditor` | String | `ShowToggle`, `EnablePageSelection`, `EnableParagraphSelection`, `EnableFileSelection`, `EnableProductSelection`, `EnableProductGroupSelection`, `DefaultLabel`, `DefaultDesign` | Link + label + style (button component) |
| `LinkEditor` | String | `EnablePageSelection`, `EnableParagraphSelection`, `SelectOnlyID`, `EnableFileSelection`, `EnableProductSelection`, `EnableProductGroupSelection` | Plain URL link (no button styling) |
| `FileEditor` | String | `Show as image selector`, `Use focal point selector for images`, `Use aspect ratio for images`, `Default aspect ratio`, `Allow file upload from frontend`, `Max files to add` | Image or file picker |
| `MediaEditor` | String | — | Video/media file picker |
| `CheckboxEditor` | Boolean | (none) | Single on/off toggle |
| `CheckboxListEditor` | String | `ShowToggle` | Multiple checkboxes from a Static options list |
| `DropDownListEditor` | String | `ShowToggle` | Single-select dropdown from a Static options list |
| `RadioButtonListEditor` | String | `ItemTemplate`, `ShowToggle` | Radio buttons from a Static options list |
| `BoxedRadioEditor` | String | (none) | Visual boxed radio buttons (S/M/L/XL size picker style) — requires generic type suffix in `type` attribute |
| `VisualSelectListEditor` | String | — | Icon-based visual option selector |
| `IntegerEditor` | String | `Min`, `Max` | Whole number input |
| `InputHTML5Editor` | String | `InputType`, `Placeholder`, `Min`, `Max`, `Step` | HTML5 input types (number, range, email, etc.) |
| `DateEditor` | DateTime | `ShowToggle` | Date/time picker |
| `ItemRelationListEditor` | String | `ItemType` (systemName of the child item type) | Ordered list of child items of a specific ItemType |
| `GeolocationEditor` | String | — | Lat/lng map picker |
| `SingleUserEditor` | String | (none) | Single user picker |
| `SingleUserGroupEditor` | String | (none) | Single user group picker |
| `ProductCatalogEditor` | String | — | Product catalog picker |
| `ProductCatalogGroupEditor` | String | — | Product catalog group picker |

#### Field examples — ready to copy

**Rich text heading (`RichTextEditorLight`):**
```xml
<field name="Title" systemName="Title" description="" type="System.String, System.Private.CoreLib" excludeFromSearch="False" defaultValueCulture="en-DK" defaultValue="My title">
  <editor type="Dynamicweb.Content.Items.Editors.RichTextEditorLight, Dynamicweb">
    <editorConfuguration><Parameters addin="Dynamicweb.Content.Items.Editors.RichTextEditorLight"><Parameter addin="Dynamicweb.Content.Items.Editors.RichTextEditorLight" name="DefaultFontStyle" value="dw-h2" /><Parameter addin="Dynamicweb.Content.Items.Editors.RichTextEditorLight" name="DefaultTag" value="h2" /><Parameter addin="Dynamicweb.Content.Items.Editors.RichTextEditorLight" name="ShowToggle" value="True" /><Parameter addin="Dynamicweb.Content.Items.Editors.RichTextEditorLight" name="ShowTagSelector" value="True" /><Parameter addin="Dynamicweb.Content.Items.Editors.RichTextEditorLight" name="ShowStyleSelector" value="True" /></Parameters></editorConfuguration>
  </editor>
</field>
```

**Rich text body (`RichTextEditor`):**
```xml
<field name="Text" systemName="Text" description="" type="System.String, System.Private.CoreLib" excludeFromSearch="False" defaultValueCulture="en-DK" defaultValue="">
  <editor type="Dynamicweb.Content.Items.Editors.RichTextEditor, Dynamicweb">
    <editorConfuguration><Parameters addin="Dynamicweb.Content.Items.Editors.RichTextEditor"><Parameter addin="Dynamicweb.Content.Items.Editors.RichTextEditor" name="DefaultFontStyle" value="dw-paragraph" /><Parameter addin="Dynamicweb.Content.Items.Editors.RichTextEditor" name="ShowToggle" value="True" /><Parameter addin="Dynamicweb.Content.Items.Editors.RichTextEditor" name="ShowTagSelector" value="True" /><Parameter addin="Dynamicweb.Content.Items.Editors.RichTextEditor" name="ShowStyleSelector" value="True" /></Parameters></editorConfuguration>
  </editor>
</field>
```

**Plain text (`TextEditor`):**
```xml
<field name="Alt text" systemName="AltText" description="" type="System.String, System.Private.CoreLib" excludeFromSearch="False" defaultValueCulture="en-DK" defaultValue="">
  <editor type="Dynamicweb.Content.Items.Editors.TextEditor, Dynamicweb">
    <editorConfuguration><Parameters addin="Dynamicweb.Content.Items.Editors.TextEditor"><Parameter addin="Dynamicweb.Content.Items.Editors.TextEditor" name="ShowToggle" value="True" /></Parameters></editorConfuguration>
  </editor>
</field>
```

**Image with focal point (`FileEditor`):**
```xml
<field name="Image" systemName="Image" description="" type="System.String, System.Private.CoreLib" excludeFromSearch="False">
  <editor type="Dynamicweb.Content.Items.Editors.FileEditor, Dynamicweb">
    <editorConfuguration><Parameters addin="Dynamicweb.Content.Items.Editors.FileEditor"><Parameter addin="Dynamicweb.Content.Items.Editors.FileEditor" name="Base directory" value="" /><Parameter addin="Dynamicweb.Content.Items.Editors.FileEditor" name="Extensions" value="" /><Parameter addin="Dynamicweb.Content.Items.Editors.FileEditor" name="Show as image selector" value="True" /><Parameter addin="Dynamicweb.Content.Items.Editors.FileEditor" name="Use focal point selector for images" value="True" /><Parameter addin="Dynamicweb.Content.Items.Editors.FileEditor" name="Allow file upload from frontend" value="False" /><Parameter addin="Dynamicweb.Content.Items.Editors.FileEditor" name="Max files to add" value="" /></Parameters></editorConfuguration>
  </editor>
</field>
```

**Button with full link options (`ButtonEditor`):**
```xml
<field name="First button" systemName="FirstButton" description="" type="System.String, System.Private.CoreLib" excludeFromSearch="False" defaultValueCulture="en-DK" defaultValue="Click here">
  <editor type="Dynamicweb.Content.Items.Editors.ButtonEditor, Dynamicweb">
    <editorConfuguration><Parameters addin="Dynamicweb.Content.Items.Editors.ButtonEditor"><Parameter addin="Dynamicweb.Content.Items.Editors.ButtonEditor" name="ShowToggle" value="True" /><Parameter addin="Dynamicweb.Content.Items.Editors.ButtonEditor" name="EnablePageSelection" value="True" /><Parameter addin="Dynamicweb.Content.Items.Editors.ButtonEditor" name="EnableParagraphSelection" value="True" /><Parameter addin="Dynamicweb.Content.Items.Editors.ButtonEditor" name="EnableFileSelection" value="True" /><Parameter addin="Dynamicweb.Content.Items.Editors.ButtonEditor" name="EnableProductSelection" value="True" /><Parameter addin="Dynamicweb.Content.Items.Editors.ButtonEditor" name="EnableProductGroupSelection" value="True" /><Parameter addin="Dynamicweb.Content.Items.Editors.ButtonEditor" name="DefaultLabel" value="Click here" /><Parameter addin="Dynamicweb.Content.Items.Editors.ButtonEditor" name="DefaultDesign" value="primary" /></Parameters></editorConfuguration>
  </editor>
</field>
```

**Checkbox (`CheckboxEditor`):**
```xml
<field name="Show title" systemName="ShowTitle" description="" type="System.Boolean, System.Private.CoreLib" excludeFromSearch="False" defaultValueCulture="en-DK" defaultValue="True">
  <editor type="Dynamicweb.Content.Items.Editors.CheckboxEditor, Dynamicweb">
    <editorConfuguration><Parameters addin="Dynamicweb.Content.Items.Editors.CheckboxEditor" /></editorConfuguration>
  </editor>
</field>
```

**Dropdown with static options (`DropDownListEditor`):**
```xml
<field name="Size" systemName="Size" description="" type="System.String, System.Private.CoreLib" excludeFromSearch="False" defaultValueCulture="en-DK" defaultValue="medium">
  <editor type="Dynamicweb.Content.Items.Editors.DropDownListEditor, Dynamicweb">
    <editorConfuguration><Parameters addin="Dynamicweb.Content.Items.Editors.DropDownListEditor"><Parameter addin="Dynamicweb.Content.Items.Editors.DropDownListEditor" name="ShowToggle" value="True" /></Parameters></editorConfuguration>
  </editor>
  <options sourceType="Static">
    <Static>
      <option name="Small" value="small" icon="" />
      <option name="Medium" value="medium" icon="" />
      <option name="Large" value="large" icon="" />
    </Static>
  </options>
</field>
```

**Boxed radio S/M/L/XL size picker (`BoxedRadioEditor`):**
```xml
<field name="Height" systemName="Height" description="" type="System.String, System.Private.CoreLib" excludeFromSearch="False" defaultValueCulture="en-DK" defaultValue="2">
  <editor type="Dynamicweb.Content.Items.Editors.BoxedRadioEditor`1, Dynamicweb">
    <editorConfuguration><Parameters addin="Dynamicweb.Content.Items.Editors.BoxedRadioEditor`1[[System.String, System.Private.CoreLib, Version=8.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e]]" /></editorConfuguration>
  </editor>
  <options sourceType="Static">
    <Static>
      <option name="S" value="1" icon="" />
      <option name="M" value="2" icon="" />
      <option name="L" value="3" icon="" />
      <option name="XL" value="4" icon="" />
    </Static>
  </options>
</field>
```

**Date picker (`DateEditor`):**
```xml
<field name="Publish date" systemName="PublishDate" description="" type="System.DateTime, System.Private.CoreLib" excludeFromSearch="False">
  <editor type="Dynamicweb.Content.Items.Editors.DateEditor, Dynamicweb">
    <editorConfuguration><Parameters addin="Dynamicweb.Content.Items.Editors.DateEditor"><Parameter addin="Dynamicweb.Content.Items.Editors.DateEditor" name="ShowToggle" value="True" /></Parameters></editorConfuguration>
  </editor>
</field>
```

**Child item list (`ItemRelationListEditor`):**
```xml
<field name="Items" systemName="Items" description="" type="System.String, System.Private.CoreLib" excludeFromSearch="False">
  <editor type="Dynamicweb.Content.Items.Editors.ItemRelationListEditor, Dynamicweb">
    <editorConfuguration><Parameters addin="Dynamicweb.Content.Items.Editors.ItemRelationListEditor"><Parameter addin="Dynamicweb.Content.Items.Editors.ItemRelationListEditor" name="ItemType" value="MyPrefix_MyBlock_Item" /></Parameters></editorConfuguration>
  </editor>
</field>
```

**Required validator** (add inside `<field>` after `</editor>`):
```xml
<validators>
  <validator type="Dynamicweb.Content.Items.Validators.RequiredValidator, Dynamicweb" />
</validators>
```

#### Rules section

```xml
<rules>
  <!-- Which websites can use this item type (* = all) -->
  <rule name="Allow in websites" type="Dynamicweb.Content.Items.Activation.AreaRestrictionRule, Dynamicweb" value="*" />

  <!-- Which parent item types can contain this block (comma-separated systemNames) -->
  <rule name="Allowed parents" type="Dynamicweb.Content.Items.Activation.ParentItemTypeRestrictionRule, Dynamicweb" value="Swift-v2_Page,Swift-v2_Row" />

  <!-- Which item types can be children of this block (empty = no restriction) -->
  <rule name="" type="Dynamicweb.Content.Items.Activation.ChildItemTypeRestrictionRule, Dynamicweb" value="" />

  <!-- Leave these three at their defaults -->
  <rule name="Allowed children" type="Dynamicweb.Content.Items.Activation.ChildRestrictionRule, Dynamicweb" value="System.Collections.Generic.HashSet`1[System.String]" />
  <rule name="Allowed parent types" type="Dynamicweb.Content.Items.Activation.ParentRestrictionRule, Dynamicweb" value="" />
  <rule name="Allow in tree sections" type="Dynamicweb.Content.Items.Activation.SectionRestrictionRule, Dynamicweb" value="" />

  <!-- Whether the block inherits the page color scheme (true = yes) -->
  <rule name="" type="Dynamicweb.Content.Items.Activation.ColorSchemeRestrictionRule, Dynamicweb" value="true" />

  <!-- Paragraphs = content block (most common); Pages = page-level item -->
  <rule name="Enable item type for" type="Dynamicweb.Content.Items.Activation.StructureRestrictionRule, Dynamicweb" value="Paragraphs" />
</rules>
```

**Common `Allowed parents` values:**

| Use case | Value |
|---|---|
| General content block | `Swift-v2_Page,Swift-v2_Row` |
| Usable everywhere (header, footer, product pages) | `Swift-v2_Footer,Swift-v2_Header,Swift-v2_Page,Swift-v2_ProductComponent,Swift-v2_ProductDetails,Swift-v2_ProductList,Swift-v2_ProductListComponent,Swift-v2_Row` |
| Child item (only inside a specific parent) | `MyPrefix_MyBlock` (the parent's systemName) |

#### Layout groups

```xml
<layout>
  <groups>
    <group name="Content" systemName="Content" sectionName="" collapsibleState="None" visibilityField="" visibilityCondition="" visibilityConditionValueType="" visibilityConditionValue="">
      <fields>
        <field systemName="Title" />
        <field systemName="Text" />
      </fields>
    </group>
    <group name="Image" systemName="Image" sectionName="" collapsibleState="None" visibilityField="" visibilityCondition="" visibilityConditionValueType="" visibilityConditionValue="">
      <fields>
        <field systemName="Image" />
        <field systemName="AltText" />
      </fields>
    </group>
    <group name="Design" systemName="Design" sectionName="" collapsibleState="None" visibilityField="" visibilityCondition="" visibilityConditionValueType="" visibilityConditionValue="">
      <fields>
        <field systemName="Size" />
      </fields>
    </group>
  </groups>
</layout>
```

To conditionally show a group based on a checkbox field: set `visibilityField` to the checkbox's `systemName` and `visibilityCondition` to `True` or `False`.

#### Corresponding Razor template

Create the template at:
```
Files/Templates/Designs/Swift-v2/Paragraph/MyPrefix_MyBlock/MyPrefix_MyBlock.cshtml
```

The template inherits from `ViewModelTemplate<ParagraphViewModel>`. Access field values via `Model.Item`:

```razor
@inherits Dynamicweb.Rendering.ViewModelTemplate<Dynamicweb.Frontend.ParagraphViewModel>

@{
    var title     = Model.Item.GetString("Title");
    var text      = Model.Item.GetString("Text");
    var image     = Model.Item.GetString("Image");
    var showTitle = Model.Item.GetBoolean("ShowTitle");
}

@if (!string.IsNullOrEmpty(title) && showTitle)
{
    <h2>@Html.Raw(title)</h2>
}
@if (!string.IsNullOrEmpty(text))
{
    <div>@Html.Raw(text)</div>
}
```

**`Model.Item` accessor methods:**

| Method | Returns | Use for |
|---|---|---|
| `GetString("fieldName")` | `string` | Text, RichText, Button, Link, Image, Dropdown — any string field |
| `GetBoolean("fieldName")` | `bool` | CheckboxEditor fields |
| `GetInt32("fieldName")` | `int` | IntegerEditor fields |
| `GetDateTime("fieldName")` | `DateTime` | DateEditor fields |
| `GetValue("fieldName")` | `object` | When the type is dynamic or unknown |

> **Note:** `Model.Item` is also available on `PageViewModel` for page-level ItemTypes. The same `GetString` / `GetBoolean` etc. methods work identically.

#### New ItemType — workflow checklist

1. Create `Files/System/Items/ItemType_MyPrefix_MyBlock.xml`
2. Create `Files/Templates/Designs/Swift-v2/Paragraph/MyPrefix_MyBlock/MyPrefix_MyBlock.cshtml`
3. Set `Allowed parents` in `<rules>` to include the Swift parent types where the block should appear
4. Restart the Dynamicweb application (or wait for an IIS recycle) — ItemTypes are discovered at startup
5. The new block appears in the paragraph picker in the admin under its `category`

> **Alternative:** ItemTypes can also be created and edited via the admin UI at **Settings → Areas → Content → Item types**. The admin writes the same XML files on disk — useful for prototyping, but committing the XML file directly is preferable for source control.

---

## Step 5 — Output format

When generating code, always:

1. **State which extension type** you are using and why
2. **Show the complete class** including namespace, using statements, and attributes
3. **Call out the NuGet package(s)** to reference
4. **Note any admin configuration steps** (e.g. "select this provider in Settings > Ecommerce > Prices")
5. **Warn about upgrade impact** if the approach requires ongoing maintenance
6. If multiple approaches are valid, **present the trade-offs** before generating

---

## Reference links

- Swift customization overview: https://doc.dynamicweb.dev/swift/customization/introduction.html
- Swift CSS design: https://doc.dynamicweb.dev/swift/customization/design-css.html
- Swift developer tools & git workflow: https://doc.dynamicweb.dev/swift/customization/developer-tools.html
- Dynamicweb extensibility overview: https://doc.dynamicweb.dev/documentation/extending/index.html
- Dynamicweb providers: https://doc.dynamicweb.dev/documentation/extending/providers.html
- Dynamicweb API reference: https://doc.dynamicweb.dev/api/
