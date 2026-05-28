---
name: xaf-environment-auth
description: >
  Configure DevExpress XAF Blazor authentication to switch between SSO (Windows Authentication) and
  standard username/password login based on ASPNETCORE_ENVIRONMENT — not Debug/Release build
  configuration. Use when setting up XAF for multi-environment deployment (Production = SSO, Test =
  password, Development = password), troubleshooting "SSO works in IIS but not in dev", or carving out
  a service-account exception for Hangfire jobs that must password-authenticate. Covers the runtime
  environment switch, appsettings.{Environment}.json layering, HangfireJob service-user pattern,
  HidePasswordLogonController, and where ASPNETCORE_ENVIRONMENT must (and must not) be set.
  Triggers: XAF SSO, XAF Windows Authentication, ASPNETCORE_ENVIRONMENT XAF, HidePasswordLogonController,
  appsettings.Production.json XAF, XAF auth per environment, HangfireJob service account.
---

# XAF Environment-Based Authentication Mode

## The Anti-Pattern: `#if DEBUG`

```csharp
// WRONG — compile-time switch
#if DEBUG
    services.AddXafAuthentication().AddPasswordAuthentication();
#else
    services.AddXafAuthentication().AddWindowsAuthentication();
#endif
```

This breaks the moment anyone builds Release locally (no domain controller → can't log in) or runs Debug in test (gets password login when policy says SSO). Build configuration is not deployment intent.

## The Pattern: Runtime Environment

`ASPNETCORE_ENVIRONMENT` decides auth mode. ASP.NET Core reads it *before* config and uses it to pick which `appsettings.{Environment}.json` to layer over `appsettings.json`.

| Environment | Auth mode | Hangfire | Source |
|---|---|---|---|
| Production | SSO; password blocked for humans | Enabled | `appsettings.Production.json` |
| Test | Username/password | Enabled | `appsettings.Test.json` |
| Development | Username/password | Disabled | `appsettings.Development.json` |

The runtime selector is config-driven, with an env-based default:

```csharp
public static class AuthenticationMode
{
    private const string UseSingleSignOnKey = "Authentication:UseSingleSignOn";

    public static bool UseSingleSignOn(IConfiguration configuration, IHostEnvironment environment)
    {
        var configured = configuration.GetValue<bool?>(UseSingleSignOnKey);
        if (configured.HasValue) return configured.Value;
        return environment.IsProduction();  // default
    }
}
```

The explicit `Authentication:UseSingleSignOn` key beats the environment default — useful for an SSO-disabled staging that runs on `ASPNETCORE_ENVIRONMENT=Production`.

## Where to Set `ASPNETCORE_ENVIRONMENT`

**Never** put it in `appsettings.json`. ASP.NET Core reads the variable before any config file is loaded, so a value inside `appsettings.json` is ignored for environment selection.

Set it via:

- **IIS**: `web.config` `<environmentVariables>` element under `<aspNetCore>`.
- **Windows services**: service environment block or `setx ASPNETCORE_ENVIRONMENT Test /M`.
- **Docker**: `ENV ASPNETCORE_ENVIRONMENT=Test` in Dockerfile or `-e ASPNETCORE_ENVIRONMENT=Test`.
- **Visual Studio**: launch profile in `Properties/launchSettings.json`.

IIS `web.config`:

```xml
<aspNetCore processPath="dotnet" arguments=".\WLNCentral.Blazor.Server.dll"
            stdoutLogEnabled="false" hostingModel="inprocess">
  <environmentVariables>
    <environmentVariable name="ASPNETCORE_ENVIRONMENT" value="Production" />
  </environmentVariables>
</aspNetCore>
```

## Wiring in Startup.cs

```csharp
public class Startup
{
    public Startup(IConfiguration configuration, IHostEnvironment env)
    {
        Configuration = configuration;
        Environment = env;
    }

    public IConfiguration Configuration { get; }
    public IHostEnvironment Environment { get; }

    public void ConfigureServices(IServiceCollection services)
    {
        var useSso = AuthenticationMode.UseSingleSignOn(Configuration, Environment);

        services.AddXafBlazor(...);

        services.AddXafSecurity(o =>
        {
            o.RoleType = typeof(PermissionPolicyRole);
            o.UserType = typeof(ApplicationUser);
        })
        .AddPasswordAuthentication(o => o.IsSupportChangePasswordPage = !useSso)
        .ConfigureAuthentication(b =>
        {
            if (useSso) b.AddWindowsAuthentication();
        });

        if (Configuration.GetValue<bool>("Jobs:UseHangfire"))
        {
            services.AddHangfire(...);
            services.AddHangfireServer();
        }
    }
}
```

Always register **password authentication**, even in Production. The Hangfire service account needs it (see below). SSO is added on top via `ConfigureAuthentication`.

## The HangfireJob Service-Account Exception

Background jobs have no Windows session, so SSO doesn't work for them. Carve out one specific username that's allowed to use password auth in Production:

```csharp
public class HidePasswordLogonController : WindowController
{
    public HidePasswordLogonController()
    {
        TargetWindowType = WindowType.Main;
    }

    protected override void OnFrameAssigned()
    {
        base.OnFrameAssigned();
        var env = ServiceProvider.GetRequiredService<IHostEnvironment>();
        var config = ServiceProvider.GetRequiredService<IConfiguration>();
        if (!AuthenticationMode.UseSingleSignOn(config, env)) return;

        Frame.GetController<LogonController>()
             .AcceptAction.Executing += (s, e) =>
        {
            var logonParams = ((LogonController)((SimpleAction)s).Controller)
                              .GetLogonParameters() as AuthenticationStandardLogonParameters;

            if (logonParams != null && logonParams.UserName != "HangfireJob")
            {
                throw new UserFriendlyException(
                    "Password sign-in is disabled. Use Windows Authentication.");
            }
        };
    }
}
```

Hangfire side authenticates `HangfireJob` via a direct `SecurityStrategyBase.Logon` call (no SignInManager because there's no Blazor circuit). The password lives at `HangfireJob:Password` in `appsettings.{Environment}.json` (read separately for each environment so dev can be blank).

```csharp
public sealed class XafJobScopeInitializer(
    IServiceProvider sp, ILogger<XafJobScopeInitializer> log) : IJobScopeInitializer
{
    private const string ServiceUserName = "HangfireJob";

    public Task InitializeAsync(CancellationToken ct = default)
    {
        var strategy = sp.GetRequiredService<ISecurityStrategyBase>();
        if (strategy.IsAuthenticated) return Task.CompletedTask;

        var nonSecured = sp.GetRequiredService<INonSecuredObjectSpaceFactory>();
        var password = sp.GetService<IConfiguration>()?["HangfireJob:Password"] ?? "";

        if (strategy is SecurityStrategy concrete)
            concrete.Authentication.SetLogonParameters(
                new AuthenticationStandardLogonParameters(ServiceUserName, password));

        using var space = nonSecured.CreateNonSecuredObjectSpace<ApplicationUser>();
        ((SecurityStrategyBase)strategy).Logon(space);
        return Task.CompletedTask;
    }
}
```

Notes:

- **`SignInManager.SignIn` does not work** in Hangfire worker threads — there is no Blazor circuit or HTTP context. Use `SecurityStrategyBase.Logon` directly.
- **`HangfireJob` must exist in the database** with the roles the jobs need (typically a custom "Service Account" role with broad type permissions but no UI navigation).
- **Do not commit the password** to source. Use environment variables or a secret store; `appsettings.Production.json` should be deployed but excluded from git.

## appsettings Layering

`appsettings.json` (base, shared defaults):

```json
{
  "Authentication": { "UseSingleSignOn": false },
  "Jobs": { "UseHangfire": false },
  "HangfireJob": { "Password": "" }
}
```

`appsettings.Production.json`:

```json
{
  "Authentication": { "UseSingleSignOn": true },
  "Jobs": { "UseHangfire": true }
}
```

`appsettings.Test.json`:

```json
{
  "Authentication": { "UseSingleSignOn": false },
  "Jobs": { "UseHangfire": true }
}
```

`appsettings.Development.json`:

```json
{
  "Authentication": { "UseSingleSignOn": false },
  "Jobs": { "UseHangfire": false }
}
```

## Verification Checklist

When auth misbehaves, check in this order:

1. **Environment**: `Get-ChildItem Env:ASPNETCORE_ENVIRONMENT` on the host (or the IIS site's env vars). The error is almost always here.
2. **Which appsettings was loaded**: log `env.EnvironmentName` and `config["Authentication:UseSingleSignOn"]` at startup.
3. **AppPool identity** (IIS): for SSO, the AppPool must run as an account that can read the user's AD token via Windows Auth — usually `ApplicationPoolIdentity` is fine if IIS Windows Authentication is enabled.
4. **IIS modules**: Windows Authentication must be installed and enabled on the site; Anonymous must be disabled for SSO endpoints.
5. **Browser**: SSO is browser-dependent. Chrome/Edge need the site in the Local Intranet zone (or an explicit allow policy) to send the Kerberos/NTLM token without prompting.

## Common Mistake

Publishing as `Debug` does **not** force classic login anymore. The build configuration only affects compile-time optimizations; what decides auth is `ASPNETCORE_ENVIRONMENT` at startup. A `Release` build running with `ASPNETCORE_ENVIRONMENT=Development` will let users log in with passwords.
