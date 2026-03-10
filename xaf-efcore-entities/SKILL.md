---
name: xaf-efcore-entities
description: >
  Author EF Core entities for DevExpress XAF applications. Use when creating, modifying, or reviewing
  XAF business objects with EF Core. Covers the critical gotchas that silently break XAF: virtual properties,
  no OwnsOne, BaseObjectInt pattern, collection initialization, decimal precision, aggregated relationships,
  DbContext registration, Web API exposure, and role permission caveats. Triggers on any XAF EF Core entity work.
---

# XAF EF Core Entity Authoring

## Entity Rules (violations fail silently)

### All properties MUST be virtual

EF Core change-tracking proxies and XAF's INotifyPropertyChanged require it. Non-virtual properties save without error but changes are silently lost.

```csharp
// CORRECT
public virtual string Name { get; set; } = string.Empty;
public virtual DateTime Date { get; set; }
public virtual ApplicationUser? User { get; set; }

// WRONG - silently fails to track changes
public string Name { get; set; } = string.Empty;
```

### No OwnsOne / owned entity types

XAF EF Core does not support `OwnsOne()`. Use separate entities with navigation properties instead.

```csharp
// CORRECT - separate entity
public virtual Address? BillingAddress { get; set; }

// WRONG - XAF will break
// modelBuilder.Entity<Client>().OwnsOne(c => c.BillingAddress);
```

### Collections: ObservableCollection with [Aggregated]

Use `ObservableCollection<T>` (not `List<T>`) for change tracking. `List<T>` silently loses collection change events.

```csharp
[Aggregated]
public virtual IList<ContactPerson> ContactPersons { get; set; } = new ObservableCollection<ContactPerson>();
```

- `[Aggregated]` = cascade delete + children cannot exist without parent
- Do NOT use `[Aggregated]` on many-to-many or optional references
- Non-aggregated nav properties use `DeleteBehavior.SetNull` by default
- When removing items in a loop, use `while (col.Count > 0) { col.Remove(col[0]); }` — never `Clear()`

### Explicit foreign key properties

Declare FK properties explicitly with `[ForeignKey]` for reliable behavior during nested detail view edits and proxy change detection.

```csharp
public virtual Guid? CustomClassId { get; set; }

[ForeignKey(nameof(CustomClassId))]
public virtual CustomClass? CustomClass { get; set; }
```

### Decimal precision in OnModelCreating

Always set precision for decimal properties or values get silently truncated.

```csharp
modelBuilder.Entity<TimeEntry>()
    .Property(p => p.Hours)
    .HasPrecision(18, 2);
```

### [NotMapped] computed properties for UI

Use `[NotMapped]` properties with `[ImmediatePostData]` for UI-friendly access (type pickers, enum dropdowns) backed by persistent string fields.

```csharp
[NotMapped]
[TypeConverter(typeof(LocalizedClassInfoTypeConverter))]
[ImmediatePostData]
public Type DataType { get; set; }  // UI-friendly

public virtual string DataTypeName { get; set; }  // Persisted
```

## BaseObjectInt Pattern

For int auto-increment keys (instead of XAF's default Guid `BaseObject`). Origin: `C:\Projects\XafSearch`.

```csharp
public abstract class BaseObjectInt : IXafEntityObject, IObjectSpaceLink
{
    protected IObjectSpace ObjectSpace;

    [Key]
    [VisibleInListView(false)]
    [VisibleInDetailView(false)]
    [VisibleInLookupListView(false)]
    public virtual int ID { get; set; }

    IObjectSpace IObjectSpaceLink.ObjectSpace
    {
        get => ObjectSpace;
        set => ObjectSpace = value;
    }

    public virtual void OnCreated() { }
    public virtual void OnSaving() { }
    public virtual void OnLoaded() { }
}
```

## Essential Attributes

| Attribute | Purpose |
|---|---|
| `[DefaultClassOptions]` | Makes entity visible in XAF navigation |
| `[DefaultProperty(nameof(Name))]` | Display text in lookups and references |
| `[Aggregated]` | Cascade delete for child collections |
| `[Required]` | Standard validation |
| `[VisibleInListView(false)]` | Hide from list views (e.g., ID column) |
| `[DomainComponent]` | Marks non-persistent objects for XAF discovery |
| `[RuleFromBoolProperty]` | Validation via computed bool properties |

## EF Core ValueConverter for unsupported types

`System.Drawing.Color` is not natively storable. Use a ValueConverter:

```csharp
public class NullableColorToInt32Converter : ValueConverter<Color?, int?>
{
    public NullableColorToInt32Converter()
        : base(c => c.HasValue ? c.Value.ToArgb() : null,
               v => v.HasValue ? Color.FromArgb(v.Value) : null) { }
}
// In OnModelCreating:
entity.Property(e => e.BackColor).HasConversion<NullableColorToInt32Converter>();
```

## Unique Indexes with GCRecord filter

For soft-delete compatibility, filter unique constraints to exclude soft-deleted rows:

```csharp
b.HasIndex(nameof(ClassName)).IsUnique().HasFilter("\"GCRecord\" = 0");
```

## Enum Storage

XAF stores enums as strings in the database (e.g., `TimeEntryStatus.Draft` becomes `"Draft"`). When bridging to external systems (e.g., SQLite in a mobile app), cast to `(int)` for storage and back to enum string for OData POST.

## DbContext Checklist

Every new entity needs:

1. `DbSet<T>` property in the DbContext
2. Decimal precision in `OnModelCreating` (if applicable)
3. Web API registration in `Startup.cs` if exposed via OData — missing = 404
4. `AdditionalExportedTypes.Add(typeof(T))` in Module if not auto-discovered

## OnModelCreating Boilerplate

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    base.OnModelCreating(modelBuilder);
    modelBuilder.UseDeferredDeletion(this);
    modelBuilder.UseOptimisticLock();
    modelBuilder.SetOneToManyAssociationDeleteBehavior(DeleteBehavior.SetNull, DeleteBehavior.Cascade);
    modelBuilder.HasChangeTrackingStrategy(
        ChangeTrackingStrategy.ChangingAndChangedNotificationsWithOriginalValues);
    modelBuilder.UsePropertyAccessMode(PropertyAccessMode.PreferFieldDuringConstruction);
    // ... entity-specific configuration
}
```

## PostgreSQL-specific

### DateTime / Npgsql UTC trap (CRITICAL)

Npgsql 6+ enforces strict `DateTimeKind` semantics. XAF generates `DateTime` values with `DateTimeKind.Unspecified` throughout its internals (audit trails, model diffs, security timestamps, and your own entities). Npgsql rejects these for `timestamp with time zone` columns with:

```
Cannot write DateTime with Kind=Unspecified to PostgreSQL type 'timestamp with time zone'
```

This affects ALL DateTime properties — your entities, XAF system tables, and internal infrastructure. The app may crash on startup before your code even runs.

**Fix:** Set the legacy switch in `Program.cs` BEFORE creating the host builder:

```csharp
AppContext.SetSwitch("Npgsql.EnableLegacyTimestampBehavior", true);
// Then: var builder = WebApplication.CreateBuilder(args);
```

This forces Npgsql to treat all timestamps as timezone-naive, matching XAF's `DateTime` semantics. Without it, even basic operations like loading security roles or writing model diffs will throw.

### Other PostgreSQL notes

- Use `ExcludeFromMigrations()` for externally-managed tables (e.g., partitioned tables)
- XAF connection strings use `EFCoreProvider=PostgreSql;` prefix — strip it when passing to non-XAF consumers (e.g., Hangfire):
  ```csharp
  var cleaned = Regex.Replace(connStr, @"EFCoreProvider\s*=\s*[^;]+;\s*", "", RegexOptions.IgnoreCase);
  ```
- XAF Model Editor always tries to load `Microsoft.EntityFrameworkCore.SqlServer` at design time, even with PostgreSQL. Add it as a design-time-only reference to avoid Model Editor errors.

## Type Storage

Store `Type.FullName` (not `AssemblyQualifiedName`) — AQN breaks on every recompilation due to assembly version changes. Resolve via `XafTypesInfo.Instance.FindTypeInfo(name)?.Type` with fallback to `Type.GetType(name)`.
