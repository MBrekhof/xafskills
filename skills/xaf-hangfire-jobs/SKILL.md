---
name: xaf-hangfire-jobs
description: >
  Integrate Hangfire background jobs into a DevExpress XAF Blazor application using a Command/Handler
  pattern that keeps the Jobs library free of XAF dependencies. Use when designing schedulable jobs,
  admin-configurable recurring jobs via a JobDefinition entity, dispatching jobs from controllers,
  authenticating Hangfire workers against XAF security, or troubleshooting startup sync failures.
  Covers the three-project layout (Jobs / Module / Blazor.Server), record commands as Hangfire
  payloads, HangfireJobDispatcher vs DirectJobDispatcher, JobSyncService startup reconciliation,
  XafJobScopeInitializer for service-account authentication, and four real gotchas: stale DateTime
  in recurring cleanup commands, ParametersJson null on startup sync, async void handlers, and
  Hangfire dashboard auth filter. Triggers: XAF Hangfire, Hangfire XAF Blazor, JobDefinition,
  IJobDispatcher, recurring job XAF, HangfireDashboardAuthFilter, XafJobScopeInitializer.
---

# XAF + Hangfire Background Jobs

## Three-Project Layout (Dependency Direction Matters)

```
YourApp.sln
├── YourApp.Module          # XAF domain — references YourApp.Jobs (commands only)
├── YourApp.Jobs            # Zero XAF dependency — commands, interfaces, dispatchers
└── YourApp.Blazor.Server   # XAF host — references both, hosts handlers + Hangfire server
```

The hard rule: **`YourApp.Jobs` never references `YourApp.Module`**. Handlers (which need domain services) live in `Blazor.Server/Handlers/`, not `Jobs/`. Module references Jobs only for the command record types — no Hangfire packages there.

Reason: Hangfire serializes job arguments to SQL Server. If commands lived in Module, Hangfire would drag XAF assemblies into its serialization graph, blow up the queue payload size, and tie job-deserialization compatibility to your XAF version.

## Commands Are Records

```csharp
namespace YourApp.Jobs.Commands;

public sealed record SendEmailCommand(
    string To, string Subject, string Body, bool IsHtml);

public sealed record CleanupCentralLogsCommand(int RetentionDays);
```

Why records:

- **JSON-serializable** out of the box; Hangfire writes them straight to its SQL Server store.
- **Immutable** — the dispatcher can't accidentally mutate a queued command.
- **Equatable** — easy to assert in unit tests.

### Gotcha: Don't put `DateTime CutoffDate` in a recurring-cleanup command

```csharp
// WRONG — admin schedules this once, the cutoff is frozen forever
public sealed record CleanupLimsLogCommand(DateTime CutoffDate);

// RIGHT — handler computes "N days ago" at execution time
public sealed record CleanupLimsLogCommand(int RetentionDays);
```

A scheduled cron job re-runs the same serialized payload on every tick. If the payload contains a hard `DateTime`, the cutoff is the date the admin saved the job — for the next decade. Use a relative interval (`int RetentionDays`) and let the handler compute `DateTime.UtcNow.AddDays(-RetentionDays)`.

This pattern only applies to **recurring** commands. One-off admin actions ("clean up everything before 2024-01-01") can take an absolute date.

## Dispatcher Abstraction

```csharp
public interface IJobDispatcher
{
    Task DispatchAsync<TCommand>(TCommand command, CancellationToken ct = default)
        where TCommand : notnull;

    void Schedule<TCommand>(TCommand command, string cronExpression)
        where TCommand : notnull;

    void Schedule<TCommand>(TCommand command, string cronExpression, string jobId)
        where TCommand : notnull;

    void RemoveSchedule(string jobId);
}
```

Two implementations:

- **`HangfireJobDispatcher`** — production. `BackgroundJob.Enqueue<JobExecutor<T>>(...)` for one-off; `RecurringJob.AddOrUpdate` for cron.
- **`DirectJobDispatcher`** — development. Resolves the handler from DI and `await`s it inline. No Hangfire server running; no SQL Server queue; fast feedback loop.

Pick at startup based on config:

```csharp
if (Configuration.GetValue<bool>("Jobs:UseHangfire"))
    services.AddSingleton<IJobDispatcher, HangfireJobDispatcher>();
else
    services.AddSingleton<IJobDispatcher, DirectJobDispatcher>();
```

`DirectJobDispatcher` makes local dev possible without standing up a Hangfire server or seeding the SQL queue schema.

## Handlers Live in Blazor.Server

```csharp
namespace YourApp.Blazor.Server.Handlers;

public class SendEmailHandler(
    IEmailService emailService,
    ILogger<SendEmailHandler> logger) : IJobHandler<SendEmailCommand>
{
    public async Task ExecuteAsync(SendEmailCommand cmd, CancellationToken ct = default)
    {
        logger.LogInformation("Sending email to {To}", cmd.To);
        try { await emailService.SendEmailAsync(cmd.To, cmd.Subject, cmd.Body, cmd.IsHtml); }
        catch (Exception ex)
        {
            logger.LogError(ex, "Failed to send email to {To}", cmd.To);
            throw;  // rethrow — Hangfire's retry policy handles it
        }
    }
}
```

Keep handlers thin: they orchestrate, they don't contain business logic. The actual work belongs in `IEmailService`, `IPlanningService`, etc., in the Module.

Always rethrow — Hangfire's automatic retry depends on the exception bubbling up. Catching and logging without rethrow makes a failed job look successful in the dashboard.

## JobExecutor — The Hangfire Entry Point

Hangfire calls one method per job; that method must:

1. Open a DI scope.
2. Authenticate the XAF security strategy (`IJobScopeInitializer`).
3. Resolve the handler.
4. Record the execution to `JobExecutionRecord` (`IJobExecutionRecorder`).
5. Invoke the handler.

```csharp
public sealed class JobExecutor<TCommand>(
    IServiceProvider rootProvider) where TCommand : notnull
{
    public async Task RunAsync(TCommand command, CancellationToken ct)
    {
        using var scope = rootProvider.CreateScope();
        var sp = scope.ServiceProvider;

        await sp.GetRequiredService<IJobScopeInitializer>().InitializeAsync(ct);

        var recorder = sp.GetRequiredService<IJobExecutionRecorder>();
        var handler = sp.GetRequiredService<IJobHandler<TCommand>>();

        var record = recorder.Start(typeof(TCommand).Name);
        try
        {
            await handler.ExecuteAsync(command, ct);
            recorder.Succeed(record);
        }
        catch (Exception ex)
        {
            recorder.Fail(record, ex);
            throw;  // bubble to Hangfire for retry
        }
    }
}
```

`HangfireJobDispatcher.DispatchAsync` enqueues:

```csharp
BackgroundJob.Enqueue<JobExecutor<TCommand>>(x => x.RunAsync(command, CancellationToken.None));
```

## XafJobScopeInitializer — Authenticating Hangfire Workers

Hangfire worker threads have no Blazor circuit and no HTTP context. `SignInManager.SignIn` does **not** work. You must call `SecurityStrategyBase.Logon` directly with a service-account credential.

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

Prerequisites:

- The `HangfireJob` user must exist in the database with the roles the jobs require.
- Password authentication must be registered in `Startup.cs` even if the app is SSO-only for humans — see `xaf-environment-auth` for the carve-out pattern.
- The password is read from `HangfireJob:Password` in `appsettings.{Environment}.json` (never committed).

## JobDefinition — Admin-Configurable Recurring Jobs

Store recurring-job metadata in a real XAF entity so admins can manage schedules through the UI.

```csharp
[DefaultClassOptions]
[NavigationItem("Background Jobs")]
public class JobDefinition
{
    [Key] public virtual Guid Id { get; set; } = Guid.NewGuid();

    [Required] [StringLength(200)]
    public virtual string Name { get; set; } = string.Empty;

    [Required] [StringLength(200)]
    [ModelDefault("PredefinedValues",
        "SendEmail;CleanupCentralLogs;CheckStalledJobs;CleanupLimsLog")]
    public virtual string JobTypeName { get; set; } = string.Empty;

    [Column(TypeName = "nvarchar(MAX)")]
    [FieldSize(FieldSizeAttribute.Unlimited)]
    public virtual string? ParametersJson { get; set; }

    [StringLength(500)]
    public virtual string? CronExpression { get; set; }

    public virtual bool IsEnabled { get; set; } = true;
    public virtual DateTime? LastRunUtc { get; set; }
    public virtual DateTime? NextRunUtc { get; set; }
    public virtual JobRunStatus LastRunStatus { get; set; } = JobRunStatus.NeverRun;

    [StringLength(500)]
    public virtual string? LastRunMessage { get; set; }

    public virtual int ConsecutiveFailures { get; set; }
}
```

Pair with a string-keyed dispatcher in `YourApp.Jobs`:

```csharp
public sealed class JobDispatchService(IJobDispatcher dispatcher, ILogger<JobDispatchService> log)
{
    public static IReadOnlyCollection<string> SupportedJobTypes { get; } = new[]
    {
        "SendEmail", "CleanupCentralLogs", "CheckStalledJobs", "CleanupLimsLog"
    };

    public async Task DispatchByNameAsync(string typeName, string? parametersJson, CancellationToken ct = default)
    {
        switch (typeName)
        {
            case "SendEmail":
                await dispatcher.DispatchAsync(Deserialize<SendEmailCommand>(parametersJson, typeName), ct);
                break;
            case "CleanupCentralLogs":
                await dispatcher.DispatchAsync(Deserialize<CleanupCentralLogsCommand>(parametersJson, typeName), ct);
                break;
            // ...
            default: throw new ArgumentException($"Unknown job type: {typeName}");
        }
    }
}
```

### Gotcha: `ParametersJson` null on startup sync

Naive deserialization (`JsonSerializer.Deserialize<T>(null!)`) throws. `JobSyncService` runs at startup and iterates every enabled `JobDefinition`; one row with null `ParametersJson` aborts the whole sync.

Defend in `Deserialize`:

```csharp
private static T Deserialize<T>(string? json, string typeName)
{
    if (string.IsNullOrWhiteSpace(json))
    {
        // Records with all-default parameters: empty JSON works.
        // Records with required parameters: this throws — caller must provide JSON.
        return JsonSerializer.Deserialize<T>("{}", JsonOptions)
            ?? throw new ArgumentException(
                $"Parameters JSON is required for job type '{typeName}'.");
    }
    return JsonSerializer.Deserialize<T>(json, JsonOptions)!;
}
```

Records like `CleanupCentralLogsCommand(int RetentionDays = 30)` deserialize from `"{}"`; commands with required parameters fail loudly at sync time so admins see "must supply ParametersJson" instead of a silent skip.

## JobSyncService — Reconciling on Startup

`IHostedService` that reads `JobDefinition` rows at startup and calls `dispatcher.Schedule(...)` to register Hangfire recurring jobs:

```csharp
public sealed class JobSyncService(
    IServiceProvider sp, ILogger<JobSyncService> log) : IHostedService
{
    private static readonly TimeSpan InitialDelay = TimeSpan.FromSeconds(10);
    private const int MaxRetries = 3;

    public async Task StartAsync(CancellationToken ct)
    {
        // Wait for XAF schema creation before reading entities
        await Task.Delay(InitialDelay, ct);

        for (int attempt = 1; attempt <= MaxRetries; attempt++)
        {
            try
            {
                using var scope = sp.CreateScope();
                var factory = scope.ServiceProvider.GetRequiredService<INonSecuredObjectSpaceFactory>();
                var dispatch = scope.ServiceProvider.GetRequiredService<JobDispatchService>();

                using var os = factory.CreateNonSecuredObjectSpace<JobDefinition>();
                var jobs = os.GetObjects<JobDefinition>().ToList();

                dispatch.SyncDefinitions(jobs.Select(j => new ScheduledJobDefinition(
                    j.JobTypeName, j.ParametersJson, j.CronExpression, j.IsEnabled)));

                foreach (var j in jobs)
                    j.NextRunUtc = dispatch.GetNextRunUtc(j.JobTypeName);
                os.CommitChanges();
                return;
            }
            catch (Exception ex) when (attempt < MaxRetries)
            {
                log.LogWarning(ex, "Job sync attempt {N} failed, retrying", attempt);
                await Task.Delay(TimeSpan.FromSeconds(5 * attempt), ct);
            }
        }
    }

    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}
```

Two important details:

- **Initial delay** — XAF schema initialization happens asynchronously during startup. Reading entities before it completes throws `Invalid object name`. Wait 10s (more in slow environments).
- **Use `INonSecuredObjectSpaceFactory`** — there's no logged-in user at host-startup, so a secured space would throw on first access.

## Dashboard Authorization

The Hangfire dashboard at `/hangfire` is unauthenticated by default. Lock it down:

```csharp
public class HangfireDashboardAuthFilter : IDashboardAuthorizationFilter
{
    public bool Authorize(DashboardContext context)
    {
        var httpContext = context.GetHttpContext();
        if (!httpContext.User.Identity?.IsAuthenticated ?? true) return false;

        var security = httpContext.RequestServices.GetService<ISecurityStrategyBase>();
        var user = security?.User as ApplicationUser;
        return user?.Roles?.Any(r => r.Name == "Administrators") == true;
    }
}

// in Startup.Configure:
app.UseHangfireDashboard("/hangfire", new DashboardOptions
{
    Authorization = new[] { new HangfireDashboardAuthFilter() }
});
```

Without this, anyone on the network can re-trigger or delete jobs. The default `LocalRequestsOnlyAuthorizationFilter` rejects requests from anywhere except `localhost` — useless in production where the user is on a different host.

## Async Void Trap

`async void` controllers/handlers swallow exceptions silently. Hangfire's retry doesn't kick in because the exception never bubbles. Use `async Task` everywhere except event handlers that are bound to non-Task-returning signatures.

```csharp
// WRONG in a controller's action handler — exceptions vanish
SomeAction.Execute += async (s, e) => { await SomeService.DoAsync(); };

// RIGHT — explicit try/catch and surface via XAF
SomeAction.Execute += async (s, e) =>
{
    try { await SomeService.DoAsync(); }
    catch (Exception ex)
    {
        Application.ShowViewStrategy.ShowMessage(ex.Message, InformationType.Error);
    }
};
```

If your refactor plan claims to eliminate `async void` but the replacement is still `async void`, you fixed nothing.

## Gotchas Summary

1. **Recurring commands must not carry absolute `DateTime`** — they freeze the cutoff.
2. **`ParametersJson` null deserialization must be defended** — `JobSyncService` iterates every row at startup and one bad row breaks all.
3. **`SignInManager.SignIn` fails in worker threads** — use `SecurityStrategyBase.Logon` directly.
4. **Job handlers must rethrow** — Hangfire's retry depends on it.
5. **Dashboard must be authorized** — default is open to anyone who hits the URL with the right host.
6. **Initial-delay the startup sync** — XAF schema isn't ready immediately.
7. **`YourApp.Jobs` never references XAF** — keep commands serializable and assembly-independent.
