---
name: xaf-conditional-appearance
description: >
  Implement data-driven conditional appearance rules in DevExpress XAF — admins create rules
  (row colors, fonts, visibility) through the UI instead of developers adding [Appearance] attributes
  to entity classes. Use when you need runtime-editable formatting rules on ListViews and DetailViews,
  or when the standard `[Appearance(...)]` attribute pattern requires too many rebuilds. Covers the
  AppearanceController.CollectAppearanceRules hook, IAppearanceRuleProperties adapter, type-prefix
  matching (FullName vs AssemblyQualifiedName), and immediate-effect cache invalidation on commit.
  Triggers: data-driven appearance, AppearanceController, CollectAppearanceRules, IAppearanceRuleProperties,
  AppearanceRulePropertiesAdapter, runtime appearance rules XAF, conditional appearance from database.
---

# XAF Data-Driven Conditional Appearance

## Why Not `[Appearance]` Attributes

XAF's built-in `[Appearance(...)]` attribute on entity classes requires a code change, rebuild, and redeployment for every formatting rule. For applications where functional administrators (not developers) need to add rules like "highlight overdue rows in red", this is too slow. Data-driven rules push appearance configuration into the database; admins manage them through the standard XAF UI; changes take effect without restart.

## Architecture

Three pieces:

1. **`AppearanceRuleData`** — persistent entity storing one rule per row (criteria, target items, colors, visibility, priority, enabled flag).
2. **`AppearanceRulePropertiesAdapter`** — implements `IAppearanceRuleProperties` so XAF's built-in appearance pipeline can consume DB rows.
3. **`DataDrivenAppearanceController`** — hooks `AppearanceController.CollectAppearanceRules` on every ObjectView to inject the adapters.

Plus a small **cache-busting controller** that resets the appearance cache when rules are saved.

## The Rule Entity

```csharp
[DefaultClassOptions]
[DefaultProperty(nameof(Name))]
public class AppearanceRuleData : IAppearanceRuleData
{
    [Key] [VisibleInDetailView(false)] [VisibleInListView(false)]
    public virtual int Id { get; set; }

    [Required] [MaxLength(256)]
    public virtual string Name { get; set; } = string.Empty;

    public virtual bool IsEnabled { get; set; } = true;

    // Stored as FullName (not AssemblyQualifiedName) to survive version bumps
    [Browsable(false)] [Required] [MaxLength(1024)]
    public virtual string DataTypeName { get; set; } = string.Empty;

    [NotMapped]
    [TypeConverter(typeof(LocalizedClassInfoTypeConverter))]
    [ImmediatePostData]
    [XafDisplayName("Target Type")]
    public Type DataType
    {
        get => string.IsNullOrWhiteSpace(DataTypeName) ? null
            : Type.GetType(DataTypeName, throwOnError: false)
              ?? XafTypesInfo.Instance.FindTypeInfo(DataTypeName)?.Type;
        set
        {
            var newName = value?.FullName ?? string.Empty;  // FullName only
            if (string.Equals(DataTypeName, newName, StringComparison.Ordinal)) return;
            DataTypeName = newName;
            Criteria = string.Empty;     // criteria depends on type — reset it
            TargetItems = string.Empty;
        }
    }

    [Browsable(false)] [MaxLength(128)]
    public virtual string Context { get; set; } = string.Empty;  // "Any", "ListView", "DetailView"

    [CriteriaOptions(nameof(DataType))]
    [MaxLength(2048)] [FieldSize(FieldSizeAttribute.Unlimited)]
    public virtual string Criteria { get; set; } = string.Empty;

    [MaxLength(2048)]
    [ModelDefault("ToolTip", "Property names separated by semicolons, or * for all properties")]
    public virtual string TargetItems { get; set; } = string.Empty;

    public virtual int Priority { get; set; }

    public virtual ViewItemVisibility? Visibility { get; set; }
    public virtual Color? BackColor { get; set; }
    public virtual Color? FontColor { get; set; }
    public virtual AppearanceFontStyle? FontStyle { get; set; }
}
```

Key choices that bite if you skip them:

- **Store `FullName`, not `AssemblyQualifiedName`** — AQN includes the assembly version which changes on every DevExpress upgrade. Save `Customer.Orders.Order`, match by prefix at runtime.
- **`[CriteriaOptions(nameof(DataType))]`** — gives the admin a XAF criteria builder targeting the chosen type. Without it, they type criteria by hand and typos blow up the view.
- **Reset `Criteria` and `TargetItems` when `DataType` changes** — otherwise admins switch the target type and end up with stale criteria that reference fields on the wrong entity.
- **Nullable `Color?`, `FontStyle?`, `Visibility?`** — null = "don't apply this aspect", so one rule can set just background color without forcing the admin to pick a font color too.

## The Adapter

XAF's appearance pipeline consumes `IAppearanceRuleProperties`. Adapt the DB row to it:

```csharp
public sealed class AppearanceRulePropertiesAdapter : IAppearanceRuleProperties
{
    public AppearanceRulePropertiesAdapter(AppearanceRuleData rule, Type targetType)
    {
        AppearanceItemType = "ViewItem";
        Context = string.IsNullOrWhiteSpace(rule.Context) ? "Any" : rule.Context;
        Criteria = rule.Criteria;
        DeclaringType = targetType;
        Method = rule.Method ?? string.Empty;
        TargetItems = rule.TargetItems ?? "*";
        BackColor = rule.BackColor;
        FontColor = rule.FontColor;
        FontStyle = rule.FontStyle.HasValue ? (DXFontStyle)(int)rule.FontStyle.Value : null;
        Visibility = rule.Visibility;
        Priority = rule.Priority;
    }
    // ... properties
}
```

`Context = "Any"` makes the rule apply to both ListView and DetailView; constrain it by setting `"ListView"` or `"DetailView"` on the rule row.

## The Controller

```csharp
public sealed class DataDrivenAppearanceController : ViewController<ObjectView>
{
    private AppearanceController appearanceController;
    private List<AppearanceRulePropertiesAdapter> cachedAdapters;

    protected override void OnActivated()
    {
        base.OnActivated();
        appearanceController = Frame.GetController<AppearanceController>();
        if (appearanceController is null) return;

        cachedAdapters = LoadAdaptersForCurrentView();

        appearanceController.ResetRulesCache();
        appearanceController.CollectAppearanceRules += OnCollectAppearanceRules;
        appearanceController.Refresh();

        ConditionalAppearanceCacheController.RulesCommitted += OnRulesCommitted;
    }

    protected override void OnDeactivated()
    {
        ConditionalAppearanceCacheController.RulesCommitted -= OnRulesCommitted;
        if (appearanceController is not null)
        {
            appearanceController.CollectAppearanceRules -= OnCollectAppearanceRules;
            appearanceController = null;
        }
        cachedAdapters = null;
        base.OnDeactivated();
    }

    private void OnCollectAppearanceRules(object s, CollectAppearanceRulesEventArgs e)
    {
        if (cachedAdapters is null) return;
        foreach (var adapter in cachedAdapters)
            e.AppearanceRules.Add(adapter);
    }

    private List<AppearanceRulePropertiesAdapter> LoadAdaptersForCurrentView()
    {
        var objectType = View.ObjectTypeInfo?.Type;
        if (objectType is null) return new();

        // Prefix-match because stored DataTypeName may include AQN with version info
        var typeFullName = objectType.FullName ?? objectType.Name;
        try
        {
            using var os = Application.CreateObjectSpace(typeof(AppearanceRuleData));
            var criteria = CriteriaOperator.Parse(
                "IsEnabled = true AND StartsWith(DataTypeName, ?)", typeFullName);
            var rules = os.GetObjects<AppearanceRuleData>(criteria);
            return rules.Select(r => new AppearanceRulePropertiesAdapter(r, objectType)).ToList();
        }
        catch { return new(); }
    }
}
```

Important details:

- **`StartsWith(DataTypeName, ?)`** prefix match handles both `Namespace.Type` and `Namespace.Type, Assembly, Version=...` storage formats. Cheap insurance against admin tooling that wrote AQN.
- **Always call `ResetRulesCache()` after subscribing** — otherwise XAF serves the cached pre-subscribe rule set and your DB rules don't show up.
- **`Frame.GetController<AppearanceController>()` may return null** for views that don't enable appearance (e.g. dashboard tiles). Guard the null and exit cleanly.
- **Catch and swallow exceptions in `LoadAdaptersForCurrentView`** — a malformed criteria stored by an admin must not prevent the view from opening. The view degrades to no rules instead of a crash.

## Cache-Busting on Commit

Without this, admins save a new rule and have to navigate away and back for it to apply.

```csharp
public sealed class ConditionalAppearanceCacheController
    : ObjectViewController<ObjectView, AppearanceRuleData>
{
    internal static event EventHandler RulesCommitted;

    protected override void OnActivated()
    {
        base.OnActivated();
        ObjectSpace.Committed += OnCommitted;
    }

    protected override void OnDeactivated()
    {
        ObjectSpace.Committed -= OnCommitted;
        base.OnDeactivated();
    }

    private void OnCommitted(object s, EventArgs e)
    {
        Frame.GetController<AppearanceController>()?.ResetRulesCache();
        Frame.GetController<AppearanceController>()?.Refresh();
        RulesCommitted?.Invoke(this, EventArgs.Empty);
    }
}
```

The static `RulesCommitted` event reaches *other* open views via `DataDrivenAppearanceController.OnRulesCommitted`. Without that, only the appearance-rule view itself refreshes; the views where rules apply stay stale until the user reopens them.

## When You Still Want `[Appearance]`

Keep `[Appearance]` attributes for rules that are:

- Tied to immutable business logic (e.g. "completed = green" is part of the spec, not configurable).
- Required at design time for tooling/preview.
- Specific to a single role's view that admins won't touch.

The data-driven controller composes additively with attribute-based rules — both contribute to `CollectAppearanceRules`. They merge by priority.

## Gotchas

- **Don't forget `ResetRulesCache()` on subscribe and unsubscribe** — XAF caches the rule list per type, and a stale cache silently hides your DB rules.
- **`DataType` setter must reset `Criteria`** — otherwise admins switch entities, the old criteria fails to parse, and the view throws at `CollectAppearanceRules` time.
- **Don't cache adapters across views** — each ObjectView gets its own controller instance, and `View.ObjectTypeInfo.Type` is what scopes the rule set.
- **Module registration** — `DataDrivenAppearanceController` must be in the shared `*.Module` project, not the platform-specific (Blazor/Win) one, so it applies to both UIs.
