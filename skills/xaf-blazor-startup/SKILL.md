---
name: xaf-blazor-startup
description: >
  Configure DevExpress XAF Blazor Server applications. Use when setting up Startup.cs, configuring middleware,
  registering modules, setting up JWT auth, OData/Web API, or troubleshooting XAF application bootstrap.
  Also use when creating non-persistent objects ([DomainComponent], NonPersistentBaseObject) — covers
  ObjectsGetting/ObjectByKeyGetting handlers, error 1021 prevention, and PopupWindowShowAction patterns.
  Covers service registration order, middleware pipeline, module lifecycle, AdditionalExportedTypes,
  DatabaseVersionMismatch handling, and Program.cs patterns. Triggers on XAF Blazor Server setup work.
---

# XAF Blazor Server Startup & Configuration

## Service Registration Order (Startup.cs)

Order matters. Follow this sequence in `ConfigureServices`:

```csharp
// 1. Basic ASP.NET services
services.AddRazorPages();
services.AddServerSideBlazor();
services.AddHttpContextAccessor();

// 2. XAF infrastructure
services.AddScoped<IAuthenticationTokenProvider, JwtTokenProviderService>();
services.AddScoped<CircuitHandler, CircuitHandlerProxy>();
services.AddSingleton(typeof(HubConnectionHandler<>), typeof(ProxyHubConnectionHandler<>));

// 3. XAF application (builder pattern)
services.AddXaf(Configuration, builder => {
    builder.UseApplication<MyBlazorApplication>();
    builder.AddXafWebApi(webApiBuilder => {
        webApiBuilder.ConfigureOptions(options => {
            options.BusinessObject<MyEntity>();  // Register each API-exposed entity
        });
    });
    builder.Modules
        .AddConditionalAppearance()
        .AddReports(options => {
            options.EnableInplaceReports = true;
            options.ReportDataType = typeof(ReportDataV2);
            options.ReportStoreMode = ReportStoreModes.XML;
        })
        .Add<MyModule>();
    builder.ObjectSpaceProviders
        .AddSecuredEFCore(options => options.PreFetchReferenceProperties())
        .WithDbContext<MyDbContext>(/* ... */)
        .AddNonPersistent();  // Required for DomainComponent/non-persistent objects
    builder.Security
        .UseIntegratedMode(options => {
            options.RoleType = typeof(PermissionPolicyRole);
            options.UserType = typeof(ApplicationUser);
            options.UserLoginInfoType = typeof(ApplicationUserLoginInfo);
        })
        .AddPasswordAuthentication();
});

// 4. Authentication (AFTER XAF)
services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie()
    .AddJwtBearer(options => { /* ... */ });

// 5. OData (AFTER authentication)
services.AddControllers().AddOData((options, sp) => {
    options.AddRouteComponents("api/odata",
        new EdmModelBuilder(sp).GetEdmModel(),
        Microsoft.OData.ODataVersion.V401,
        rs => rs.ConfigureXafWebApiServices())
    .EnableQueryFeatures(100);
});
```

**Key points:**
- `PreFetchReferenceProperties()` — critical performance optimization for security evaluation
- `AddNonPersistent()` — required if using `[DomainComponent]` non-persistent objects
- `UsedExportedTypes.Custom` in Module constructor prevents SecurityModule from auto-including all types

## JSON Serialization

```csharp
services.Configure<JsonOptions>(o => {
    o.JsonSerializerOptions.PropertyNamingPolicy = null;  // Exact case matching for OData
});
```

Without this, OData query binding fails silently due to case mismatch.

## HTTP Pipeline Order (Configure method)

```csharp
app.UseExceptionHandler(...);
app.UseHttpsRedirection();   // See gotcha below for monitoring endpoints
app.UseRequestLocalization();
app.UseStaticFiles();
app.UseAuthentication();     // MUST be before Authorization
app.UseAuthorization();      // MUST be before UseXaf
app.UseAntiforgery();
app.UseXaf();                // MUST be after auth/authz
app.MapXafEndpoints();       // Before MapBlazorHub
app.MapBlazorHub();
```

`UseXaf()` after auth is mandatory — XAF needs the security context.

### HTTPS redirect blocks Prometheus/health check scraping

`UseHttpsRedirection()` returns 307 for ALL HTTP requests, including `/metrics` and `/health`. Docker containers (Prometheus, load balancers) scrape these over HTTP and can't follow the redirect to HTTPS with a dev cert. Exclude monitoring endpoints:

```csharp
app.UseWhen(
    context => !context.Request.Path.StartsWithSegments("/metrics")
            && !context.Request.Path.StartsWithSegments("/health"),
    appBuilder => appBuilder.UseHttpsRedirection());
```

## Module Lifecycle

### Module.Setup() runs BEFORE XAF model generation

Register types here if they must be in the model:

```csharp
public override void Setup(XafApplication application)
{
    // Register runtime types BEFORE base.Setup()
    XafTypesInfo.Instance.RegisterEntity(myType);
    AdditionalExportedTypes.Add(myType);

    base.Setup(application);  // Model generation happens here
}
```

### EarlyBootstrap for Web API timing

`AddXafWebApi()` runs during `ConfigureServices`, before `Module.Setup()`. If you need runtime types available for Web API, compile them in a static `EarlyBootstrap()` called before `services.AddXaf()`.

### AdditionalExportedTypes

Entities not auto-discovered by XAF must be registered explicitly:

```csharp
public MyModule()
{
    AdditionalExportedTypes.Add(typeof(ApplicationUser));
    AdditionalExportedTypes.Add(typeof(PermissionPolicyRole));
    RequiredModuleTypes.Add(typeof(SecurityModule));
    RequiredModuleTypes.Add(typeof(ConditionalAppearanceModule));
}
```

Forgetting `AdditionalExportedTypes` = entity invisible in navigation (silent failure).

## DatabaseVersionMismatch Handling

```csharp
// BlazorApplication.cs
void OnDatabaseVersionMismatch(object sender, DatabaseVersionMismatchEventArgs e)
{
    if (Debugger.IsAttached) {
        e.Updater.Update();
        e.Handled = true;
    } else {
        throw new InvalidOperationException("Database schema mismatch");
    }
}
```

For dynamic type scenarios, always auto-update: `e.Updater.Update(); e.Handled = true;`

## Program.cs Patterns

```csharp
// Set BEFORE host creation
DevExpress.ExpressApp.FrameworkSettings.DefaultSettingsCompatibilityMode
    = FrameworkSettingsCompatibilityMode.Latest;
DevExpress.ExpressApp.Security.SecurityStrategy.AutoAssociationReferencePropertyMode
    = ReferenceWithoutAssociationPermissionsMode.AllMembers;

// Database update via CLI
if (args.Contains("--updateDatabase")) { /* ... */ }

// PostgreSQL: set before anything else
AppContext.SetSwitch("Npgsql.EnableLegacyTimestampBehavior", true);
```

## JWT Token Provider

```csharp
public class JwtTokenProviderService : IAuthenticationTokenProvider
{
    public string Authenticate(object logonParameters)
    {
        var result = signInManager.AuthenticateByLogonParameters(logonParameters);
        if (!result.Succeeded)
            throw new AuthenticationException("Invalid credentials");

        var token = new JwtSecurityToken(
            claims: result.Principal.Claims,  // Claims from SignInManager, not manual
            expires: DateTime.Now.AddDays(2),
            signingCredentials: new SigningCredentials(key, SecurityAlgorithms.HmacSha256));
        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

## OData Client Gotchas

- `@odata.bind` syntax for navigation properties: `"ProjectTask@odata.bind": "ProjectTask(5)"` — posting foreign key integers directly fails silently
- Response wrapper: OData returns `{ "value": [...] }`, not bare arrays
- OData V401 must match between server config and client expectations

## Diagnostic Info Action

Shows all active controllers, actions, and validation rules for the current view. Essential for debugging why an action or controller is inactive.

In `appsettings.Development.json`:

```json
{
  "DevExpress": {
    "ExpressApp": {
      "EnableDiagnosticActions": true
    }
  }
}
```

Adds a **Diagnostic** menu to the toolbar with Actions Info, Controllers Info, and View Info.

## Non-Persistent Objects

For UI-only entities (parameter forms, search DTOs, selection screens). Requires `AddNonPersistent()` in ObjectSpaceProviders.

### Base class is mandatory

Plain POCOs fail silently. Always inherit `NonPersistentBaseObject` (provides `Oid` key + `IObjectSpaceLink`):

```csharp
[DomainComponent]
[DefaultClassOptions]
[NavigationItem("My Group")]
public class MyParameters : NonPersistentBaseObject
{
    public virtual string Name { get; set; }
}
```

Register in module constructor: `AdditionalExportedTypes.Add(typeof(MyParameters));`

### ObjectsGetting + ObjectByKeyGetting (CRITICAL)

Without both handlers, you get **error 1021**: "object belongs to another ObjectSpace." `ObjectsGetting` populates ListViews. `ObjectByKeyGetting` resolves objects when opening DetailViews from a ListView row click.

Wire both via a `WindowController` that hooks `Application.ObjectSpaceCreated`:

```csharp
public class MyNonPersistentController : WindowController
{
    protected override void OnActivated()
    {
        base.OnActivated();
        Application.ObjectSpaceCreated += Application_ObjectSpaceCreated;
    }

    protected override void OnDeactivated()
    {
        Application.ObjectSpaceCreated -= Application_ObjectSpaceCreated;
        base.OnDeactivated();
    }

    private void Application_ObjectSpaceCreated(object sender, ObjectSpaceCreatedEventArgs e)
    {
        if (e.ObjectSpace is not NonPersistentObjectSpace npOs) return;
        npOs.ObjectsGetting += NpOs_ObjectsGetting;
        npOs.ObjectByKeyGetting += NpOs_ObjectByKeyGetting;
    }

    private void NpOs_ObjectsGetting(object sender, ObjectsGettingEventArgs e)
    {
        if (e.ObjectType != typeof(MyParameters)) return;
        var os = (NonPersistentObjectSpace)sender;
        var instance = os.CreateObject<MyParameters>();
        // Set default values here
        e.Objects = new ArrayList { instance };
    }

    private void NpOs_ObjectByKeyGetting(object sender, ObjectByKeyGettingEventArgs e)
    {
        if (e.ObjectType != typeof(MyParameters)) return;
        var os = (NonPersistentObjectSpace)sender;
        e.Object = os.CreateObject<MyParameters>();
    }
}
```

**Key rule:** Always create objects with `os.CreateObject<T>()` on the **current** ObjectSpace — never cache and reuse objects across ObjectSpace instances.

### When to use PopupWindowShowAction instead

`[DefaultClassOptions]` creates a ListView nav item → requires ObjectsGetting/ObjectByKeyGetting. For one-shot parameter forms (trigger action → show form → process → close), use `PopupWindowShowAction` instead — simpler, no ObjectSpace wiring needed:

```csharp
private void Action_CustomizePopup(object sender, CustomizePopupWindowParamsEventArgs e)
{
    var os = Application.CreateObjectSpace(typeof(MyParameters));
    var param = os.CreateObject<MyParameters>();
    e.View = Application.CreateDetailView(os, param, true);
    e.DialogController.SaveOnAccept = false;
}
```

### Accessing persistent data from non-persistent views

When a non-persistent object's controller needs to read/write persistent entities, create a **separate** ObjectSpace:

```csharp
IObjectSpace persistentOs = Application.CreateObjectSpace(typeof(Customer));
try
{
    var customers = persistentOs.GetObjects<Customer>();
    // ... work with persistent data ...
}
finally
{
    persistentOs.Dispose();
}
```

Never mix persistent objects from one ObjectSpace with the non-persistent View's ObjectSpace. After modifying non-persistent objects, call `View.ObjectSpace.SetModified(obj)` and `View.Refresh()` to update the UI.

### Editable Popup ListView for Non-Persistent Objects

When showing non-persistent objects in a popup ListView with inline editing (e.g., checkbox toggles), several gotchas apply:

```csharp
// 1. Lock the BindingList — prevents DxGrid E1057 "cannot create/delete rows"
var items = new BindingList<MySelection> { AllowNew = false, AllowRemove = false };

// 2. Populate items via NonPersistentObjectSpace.ObjectsGetting
if (os is NonPersistentObjectSpace npOs)
{
    npOs.ObjectsGetting += (s, args) => {
        if (args.ObjectType == typeof(MySelection))
            args.Objects = items;
    };
}

// 3. Create ListView with Client data access mode (only option for in-memory data)
var collectionSource = Application.CreateCollectionSource(
    os, typeof(MySelection), listViewId,
    CollectionSourceDataAccessMode.Client, CollectionSourceMode.Normal);
var listView = Application.CreateListView(listViewId, collectionSource, false);

// 4. Enable editing — must override model default AND security system
listView.Model.AllowEdit = true;
listView.AllowEdit.SetItemValue("Info.AllowEdit", true);  // Security blocks DomainComponents
listView.AllowEdit.SetItemValue("MyPopup", true);
listView.AllowNew.SetItemValue("MyPopup", false);         // Hide New action
listView.AllowDelete.SetItemValue("MyPopup", false);       // Hide Delete action

// 5. Set edit mode via reflection (Module project can't reference Blazor types)
//    "Batch" = cell-level editing, click checkbox directly
//    "Inline" = row-level, requires Edit button (won't work without command column)
TrySetEnumProperty(listView.Model, "InlineEditMode", "Batch");
TrySetBooleanProperty(listView.Model, "ShowSelectionColumn", false);

// 6. Configure popup dialog
e.View = listView;
e.DialogController.SaveOnAccept = false;
e.DialogController.CloseOnCurrentObjectProcessing = false;

// 7. Disable detail view navigation on row click
e.DialogController.Activated += (s, _) => {
    var ctrl = ((Controller)s!).Frame
        .GetController<ListViewProcessCurrentObjectController>();
    if (ctrl != null) ctrl.Active["MyPopup"] = false;
};
```

**InlineEditMode values** (`IModelListViewBlazor.InlineEditMode`):

| Value | Behavior |
|-------|----------|
| `Inline` (default) | Edit button required — won't work without command column |
| `Batch` | Click any cell directly — best for checkbox toggles |
| `EditForm` | Inline form replaces the row |
| `PopupEditForm` | Popup form for the row |

**Reflection helpers** (needed because Module can't reference Blazor assembly):

```csharp
static bool TrySetBooleanProperty(object? target, string name, bool value)
{
    var prop = target?.GetType().GetProperty(name);
    if (prop?.CanWrite != true || prop.PropertyType != typeof(bool)) return false;
    prop.SetValue(target, value);
    return true;
}

static bool TrySetEnumProperty(object? target, string name, string valueName)
{
    var prop = target?.GetType().GetProperty(name);
    if (prop?.CanWrite != true || !prop.PropertyType.IsEnum) return false;
    if (!Enum.GetNames(prop.PropertyType).Contains(valueName)) return false;
    prop.SetValue(target, Enum.Parse(prop.PropertyType, valueName));
    return true;
}
```
