---
name: xaf-security
description: >
  Configure DevExpress XAF security: roles, permissions, users, and background job authentication.
  Use when setting up role permissions in Updater.cs, implementing object-level security, exporting/importing
  roles, creating background job security contexts, or troubleshooting permission issues. Covers type-level
  vs object-level permissions, CurrentUserIdOperator, PermissionsReloadMode, and the role update caveat.
---

# XAF Security & Permissions

## Two-Level Permission System

Both levels must allow access for a user to see data:

1. **Type-level**: "Can this role read Client entities?" — `AddTypePermissionsRecursively<Client>(SecurityOperations.Read)`
2. **Object-level**: "Can this user read THIS specific TimeEntry?" — `AddObjectPermissionFromLambda<TimeEntry>(...)`

If type-level denies, object-level is never checked.

## Setting Permissions in Updater.cs

### Type-level (broad access)

```csharp
role.AddTypePermissionsRecursively<Client>(SecurityOperations.ReadWriteAccess, SecurityPermissionState.Allow);
```

### Object-level with lambda (owner filtering)

```csharp
role.AddObjectPermissionFromLambda<TimeEntry>(
    SecurityOperations.ReadWriteAccess,
    t => t.User!.ID == (Guid)CurrentUserIdOperator.CurrentUserId(),
    SecurityPermissionState.Allow);
```

### Object-level with string criteria

```csharp
role.AddObjectPermission<Project>(
    SecurityOperations.Read,
    "ProjectAssignments[User.ID = CurrentUserId()]",
    SecurityPermissionState.Allow);
```

### Member-level (specific properties)

```csharp
role.AddMemberPermissionFromLambda<ApplicationUser>(
    SecurityOperations.Write, "StoredPassword",
    u => u.ID == (Guid)CurrentUserIdOperator.CurrentUserId(),
    SecurityPermissionState.Allow);
```

## Role Permissions Do NOT Auto-Update

XAF creates roles on first run with `if (role == null)` checks. If the role already exists, code changes to permissions are **ignored**. After modifying permissions in Updater.cs:
- Drop and recreate the database, OR
- Update permissions manually via XAF Blazor admin UI

## PermissionsReloadMode

```csharp
((SecurityStrategy)securityStrategy).PermissionsReloadMode = PermissionsReloadMode.NoCache;
```

- `NoCache` (default): Permissions checked fresh on every access. Required for live role switching. Higher DB load.
- `CacheOnFirstAccess`: Cached per session. Better performance but changes require re-login.

## Seed Data Pattern

```csharp
#if !RELEASE
    var adminRole = ObjectSpace.FirstOrDefault<PermissionPolicyRole>(r => r.Name == "Administrators");
    if (adminRole == null)
    {
        adminRole = ObjectSpace.CreateObject<PermissionPolicyRole>();
        adminRole.Name = "Administrators";
        adminRole.IsAdministrative = true;  // Bypasses ALL permission checks
    }
    // Create test users with known passwords for dev
    var admin = ObjectSpace.FirstOrDefault<ApplicationUser>(u => u.UserName == "Admin");
    if (admin == null) { /* create user, assign role */ }
    ObjectSpace.CommitChanges();
#endif
```

## Role Export/Import

### Type resolution (three-tier fallback)

When importing roles from JSON, resolve type names with:

```csharp
var type = XafTypesInfo.Instance.FindTypeInfo(typeName)?.Type  // XAF type system
        ?? Type.GetType(typeName)                               // Standard .NET
        ?? ScanAssemblies(typeName);                            // Assembly scan (last resort)
```

Store `Type.FullName` (not `AssemblyQualifiedName`) — AQN breaks on every recompilation.

### Nested permission clearing order

When updating existing roles, delete children before parents:

```csharp
while (role.TypePermissions.Count > 0) {
    var tp = role.TypePermissions[0];
    while (tp.ObjectPermissions.Count > 0) objectSpace.Delete(tp.ObjectPermissions[0]);
    while (tp.MemberPermissions.Count > 0) objectSpace.Delete(tp.MemberPermissions[0]);
    objectSpace.Delete(tp);
}
```

## Delete Permission Gotcha

If users need to unpin/remove items, the role must include Delete:

```csharp
role.AddTypePermissionsRecursively<UserPreference>(
    SecurityOperations.ReadWriteAccess + ";" + SecurityOperations.Delete,
    SecurityPermissionState.Allow);
```

Missing Delete causes "prohibited by security rules" on remove operations.

## Background Job Security Context

Hangfire/background jobs run with no authenticated user. XAF throws `UserFriendlySecurityException: The user name must not be empty`.

**Solution**: Create a service user and authenticate in the job scope:

```csharp
public class XafJobScopeInitializer : IJobScopeInitializer
{
    public void Initialize(IServiceProvider scopeProvider)
    {
        var userManager = scopeProvider.GetRequiredService<UserManager>();
        var signInManager = scopeProvider.GetRequiredService<SignInManager>();
        var user = userManager.FindUserByName("HangfireJob");
        signInManager.SignIn(user);
    }
}
```

- Must run **inside** the service scope (UserManager/SignInManager are scoped)
- Service user needs a minimal role with read-only access to required types
- For operations that bypass security entirely, use `INonSecuredObjectSpaceFactory`

## Non-Secured ObjectSpace

For background operations that should bypass security (audit recording, AI tools):

```csharp
var scope = serviceProvider.CreateScope();
var factory = scope.ServiceProvider.GetRequiredService<INonSecuredObjectSpaceFactory>();
var os = factory.CreateNonSecuredObjectSpace(entityType);
```

Always dispose both the ObjectSpace and the scope.

## Hangfire + XAF DbContext Incompatibility

XAF's `ChangingAndChangedNotificationsWithOriginalValues` change tracking strategy is incompatible with Hangfire background threads. Use raw `SqlConnection` for data access in Hangfire jobs, not the XAF DbContext.

## ApplicationUser Pattern

```csharp
[DefaultProperty(nameof(UserName))]
public class ApplicationUser : PermissionPolicyUser,
    ISecurityUserWithLoginInfo, ISecurityUserLockout
{
    public virtual int AccessFailedCount { get; set; }
    public virtual DateTime LockoutEnd { get; set; }

    [Browsable(false)]
    [Aggregated]
    public virtual IList<ApplicationUserLoginInfo> UserLogins { get; set; }
        = new ObservableCollection<ApplicationUserLoginInfo>();
}
```

- Inherit `PermissionPolicyUser` (not custom base)
- `ISecurityUserWithLoginInfo` enables OAuth
- `ISecurityUserLockout` enables password lockout
- `ApplicationUserLoginInfo` type must be registered in Security config
