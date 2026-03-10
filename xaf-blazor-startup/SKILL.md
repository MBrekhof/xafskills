---
name: xaf-blazor-startup
description: >
  Configure DevExpress XAF Blazor Server applications. Use when setting up Startup.cs, configuring middleware,
  registering modules, setting up JWT auth, OData/Web API, or troubleshooting XAF application bootstrap.
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
app.UseHttpsRedirection();
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

## Non-Persistent Objects

For UI-only entities (chat panels, parameter forms, search DTOs):

```csharp
[DomainComponent]
[DefaultClassOptions]
public class MyParameters : NonPersistentBaseObject { }
```

Populate via `NonPersistentObjectSpace.ObjectsGetting` event:

```csharp
npOs.ObjectsGetting += (s, args) => {
    if (args.ObjectType == typeof(MyParameters))
        args.Objects = myItems;
};
```

Requires `AddNonPersistent()` in ObjectSpaceProviders.
