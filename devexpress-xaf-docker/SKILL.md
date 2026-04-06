---
name: devexpress-xaf-docker
description: >
  Deploy DevExpress XAF Blazor Server apps in Docker containers and handle XAF EF Core provider
  gotchas. Use when building Dockerfiles for XAF, DevExpress Blazor, or any .NET app using
  DevExpress.Drawing.Skia/SkiaSharp in Linux containers. Also use when setting up XAF with
  non-SqlServer databases (PostgreSQL, MySQL, etc.) or fixing Model Editor errors.
  Triggers: Dockerfile for XAF, DevExpress in Docker, SkiaSharp Docker, libSkiaSharp error,
  libfontconfig error, XAF container, DevExpress container deployment, XAF PostgreSQL,
  XAF Model Editor error, SqlServer assembly not found, EFCoreProvider=Postgres.
---

# DevExpress XAF Docker & EF Core Provider Guide

## SkiaSharp Native Dependencies

The `dotnet/aspnet:8.0` image lacks native libraries SkiaSharp needs. The NuGet package `SkiaSharp.NativeAssets.Linux` provides the managed wrapper but NOT the system dependencies.

**Install in the runtime stage of the Dockerfile:**

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
RUN apt-get update && apt-get install -y --no-install-recommends \
    libfontconfig1 libfreetype6 && \
    rm -rf /var/lib/apt/lists/*
```

Without this: `DllNotFoundException: Unable to load shared library 'libSkiaSharp'` / `libfontconfig.so.1: cannot open shared object file`.

## SkiaSharp Version Must Match DevExpress

DevExpress.Drawing.Skia pins a specific SkiaSharp version. Mismatches cause runtime failures.

| DevExpress | SkiaSharp.NativeAssets.Linux |
|------------|----------------------------|
| 25.2.x     | 3.119.1                    |
| 24.2.x     | 2.88.9                     |

Check with: `dotnet list package --include-transitive | grep SkiaSharp`

## XAF Database Initialization

XAF must create its schema before serving pages. Run after container starts:

```bash
docker compose exec <service> dotnet <assembly>.dll --updateDatabase --forceUpdate --silent
```

Without this, XAF shows "Application Error" in the browser and any EF/SQL queries against XAF tables fail with `Invalid object name`.

Consider automating this as an entrypoint script or init container.

## .dockerignore is Critical on Windows

Windows `bin/` and `obj/` directories copied into a Linux build container cause failures with `--no-restore` (wrong platform binaries). Always include:

```
bin/
obj/
.vs/
*.user
node_modules/
```

## Multi-Stage Dockerfile Pattern

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
RUN apt-get update && apt-get install -y --no-install-recommends \
    libfontconfig1 libfreetype6 && \
    rm -rf /var/lib/apt/lists/*
WORKDIR /app
EXPOSE 5001

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY nuget.config .
COPY MyProject.Module/MyProject.Module.csproj MyProject.Module/
COPY MyProject.Blazor.Server/MyProject.Blazor.Server.csproj MyProject.Blazor.Server/
RUN dotnet restore MyProject.Blazor.Server/MyProject.Blazor.Server.csproj
COPY . .
RUN dotnet publish MyProject.Blazor.Server/MyProject.Blazor.Server.csproj -c Release -o /app/publish --no-restore

FROM base AS final
WORKDIR /app
COPY --from=build /app/publish .
ENV ASPNETCORE_URLS=http://+:5001
ENTRYPOINT ["dotnet", "MyProject.Blazor.Server.dll"]
```

Key points:
- `nuget.config` copied before restore (contains DevExpress private feed)
- `nuget.config` should be gitignored (contains personal feed URL)
- Native deps installed in base stage, inherited by final stage
- `--no-restore` in publish since restore already ran

## XAF EF Core with Non-SqlServer Databases

### Model Editor Requires SqlServer Assembly

XAF's Visual Studio Model Editor uses `DevExpress.ExpressApp.EFCore.DesignTime.DesignTimeDbContextCreator` which **always** tries to load `Microsoft.EntityFrameworkCore.SqlServer`, even when the project uses a different provider (PostgreSQL, MySQL, etc.).

**Fix:** Add SqlServer as a dependency even though it's not used at runtime:

```xml
<PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="8.0.18" />
```

Without this: `System.TypeLoadException: Could not find assembly 'Microsoft.EntityFrameworkCore.SqlServer'` when opening the Model Editor.

Note: Adding `IDesignTimeDbContextFactory<TContext>` does NOT help — XAF's design-time tooling ignores it and uses its own creator.

### PostgreSQL Provider Setup

For XAF + EF Core + PostgreSQL:

1. **Connection string prefix:** Use `EFCoreProvider=Postgres` so XAF auto-selects Npgsql:
   ```
   EFCoreProvider=Postgres;Host=localhost;Port=5432;Database=mydb;Username=user;Password=pass
   ```

2. **Legacy timestamp behavior:** Add before `CreateHostBuilder` in Program.cs:
   ```csharp
   AppContext.SetSwitch("Npgsql.EnableLegacyTimestampBehavior", true);
   ```
   Without this: `Cannot write DateTime with Kind=Utc to PostgreSQL type 'timestamp without time zone'`.

3. **Required packages:**
   ```xml
   <PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="8.0.*" />
   <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="8.0.18" />
   ```

## Container-to-Host Communication

Docker containers reach the host via `host.docker.internal` (Docker Desktop only). Key gotchas:

- **Kestrel only** — IIS Express binds to `localhost` and is unreachable from containers. Use the Kestrel launch profile (`dotnet run` or VS "Project" profile).
- **HTTPS redirect breaks scraping** — `UseHttpsRedirection()` redirects container HTTP requests to HTTPS with a dev cert the container can't verify. Exclude monitoring endpoints with `app.UseWhen()` (see xaf-blazor-startup skill).
- **Prometheus config**: use `host.docker.internal:<kestrel-http-port>` with `scheme: http`.

### XAF Does NOT Support Composite Keys with EF Core

XAF requires a single-column primary key. For databases that need composite keys (e.g., PostgreSQL partitioned tables with `PRIMARY KEY (id, partition_key)`):

- EF Core uses a single column (e.g., `id BIGSERIAL`) as the sole key for object identity/tracking
- The database maintains the composite PK for its own purposes (partitioning, etc.)
- This works as long as the single column is globally unique (BIGSERIAL/IDENTITY)

### Externally-Managed Tables (Partitioned Tables, Pre-Existing Schema)

When tables are managed outside EF Core (SQL scripts, partitioned tables, etc.), XAF's schema updater will try to drop/recreate PKs and FKs to match the EF model — which fails on partitioned tables.

**Fix:** Use `ExcludeFromMigrations()` + `UpdateDatabaseAlways`:

1. In `OnModelCreating`, exclude externally-managed tables from migrations:
   ```csharp
   modelBuilder.Entity<Ticket>(b =>
   {
       b.ToTable("tickets", t => t.ExcludeFromMigrations());
       b.HasKey(t => t.Id);
       // ... other config
   });
   ```

2. In `BlazorApplication`, always run the updater (so XAF system tables get created):
   ```csharp
   protected override void OnSetupStarted()
   {
       base.OnSetupStarted();
       DatabaseUpdateMode = DatabaseUpdateMode.UpdateDatabaseAlways;
   }
   ```

Without `ExcludeFromMigrations`: `cannot drop constraint X_pkey on table X because other objects depend on it`.
Without `UpdateDatabaseAlways`: XAF system tables (security, model diffs) won't be created on first run.
