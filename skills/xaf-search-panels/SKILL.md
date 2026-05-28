---
name: xaf-search-panels
description: >
  Build configurable advanced-search panels in DevExpress XAF where admins define searchable fields
  through the UI and the project regenerates C# search DTOs at build time. Use when adding per-entity
  search popups, criteria-builder panels, or "require search before load" guards on heavy list views.
  Covers the critical gotcha that runtime Roslyn compilation of search DTOs is incompatible with
  AddSecuredEFCore, the [DomainComponent] non-persistent DTO pattern, SearchControllerBase<TEntity,TDTO>,
  CriteriaBuilder, wildcard support, and the PopupWindowShowAction + NonPersistentObjectSpace flow.
  Triggers: XAF advanced search, search panel, DomainComponent search DTO, NonPersistentObjectSpace
  popup, PopupWindowShowAction search, CriteriaBuilder XAF, AddSecuredEFCore dynamic types.
---

# XAF Configurable Search Panels

## The Critical Gotcha: Runtime Roslyn vs AddSecuredEFCore

Do **not** compile search DTOs at runtime with Roslyn. `AddSecuredEFCore` creates change-tracking proxies for every registered entity type at startup, and types compiled later cannot participate in that proxy infrastructure. The secured data layer never sees runtime-compiled types and `CreateObject<TSearchDTO>()` silently returns an unproxied instance that doesn't round-trip through the UI.

**Correct approach**: admins configure search panels through the XAF UI (rows in a `SearchConfiguration` / `SearchField` entity). A "Save to Project" action generates plain `.cs` files at design time. The solution is rebuilt and redeployed. Generated DTOs compile normally and participate in EF Core change tracking.

This trades runtime configurability for security-system compatibility. The trade-off is forced — there is no working runtime path.

## DTO Pattern (Generated)

Search DTOs are **non-persistent** objects marked `[DomainComponent]`. They live alongside generated search controllers.

```csharp
[DomainComponent]
public class BatchZoeker
{
    // Blazor non-persistent editors need a stable key so values round-trip
    // correctly from the popup back into the DTO instance.
    [Key]
    [Browsable(false)]
    [VisibleInDetailView(false)]
    [VisibleInListView(false)]
    public Guid Id { get; set; } = Guid.NewGuid();

    [VisibleInDetailView(true), VisibleInListView(false)]
    [XafDisplayName("Name")]
    [ImmediatePostData]
    [ToolTip("Supports wildcards: * (any chars), ? (single char)")]
    public virtual string Name { get; set; }

    [VisibleInDetailView(true), VisibleInListView(false)]
    [XafDisplayName("BatchStatus")]
    [ImmediatePostData]
    [UseExactMatch]
    public virtual string BatchStatus { get; set; }

    [VisibleInDetailView(true), VisibleInListView(false)]
    [XafDisplayName("DateCreated (From)")]
    [ImmediatePostData]
    public virtual DateTime? DateCreatedFrom { get; set; }

    [VisibleInDetailView(true), VisibleInListView(false)]
    [XafDisplayName("DateCreated (To)")]
    [ImmediatePostData]
    public virtual DateTime? DateCreatedTo { get; set; }
}
```

Rules that quietly break otherwise:

- **`[Key] Guid Id`** — Blazor non-persistent editors lose values on round-trip without a stable key. Mark it `Browsable(false)` so admins don't see it.
- **`virtual` properties** — required even for non-persistent DTOs because XAF wires up change notifications via proxies the same way it does for entities.
- **Nullable types for ranges** — `DateTime?`, `decimal?` so "not set" is distinguishable from "zero / epoch".
- **`[ImmediatePostData]`** — pushes the value back to the model on each keystroke; without it the user must tab out before the search action sees the value.
- **`[UseExactMatch]`** — custom attribute the `CriteriaBuilder` uses to skip wildcard expansion for enum-like columns where `LIKE` is wrong.

## SearchControllerBase

One generic base controller does all the work; each entity gets a tiny derived class.

```csharp
public abstract class SearchControllerBase<TEntity, TSearchDTO>
    : ObjectViewController<ListView, TEntity>
    where TEntity : class
    where TSearchDTO : class
{
    private const string CriteriaKey = "RuntimeAdvancedSearch";
    private const string RequireSearchKey = "RequireSearchBeforeLoad";
    protected virtual int MaxActiveFilters { get; set; } = 20;
    protected virtual bool RequireSearchBeforeLoad => true;

    private PopupWindowShowAction searchAction;

    public SearchControllerBase()
    {
        searchAction = new PopupWindowShowAction(this,
            $"Search_{typeof(TEntity).Name}", PredefinedCategory.View)
        {
            Caption = "Advanced Search",
            ImageName = "Action_Search",
            SelectionDependencyType = SelectionDependencyType.Independent
        };
        searchAction.CustomizePopupWindowParams += CustomizePopup;
        searchAction.Execute += Execute;
    }

    protected override void OnActivated()
    {
        base.OnActivated();
        if (RequireSearchBeforeLoad)
            View.CollectionSource.Criteria[RequireSearchKey] = CriteriaOperator.Parse("1=0");
    }

    private void CustomizePopup(object s, CustomizePopupWindowParamsEventArgs e)
    {
        var os = (NonPersistentObjectSpace)Application.CreateObjectSpace(typeof(TSearchDTO));
        os.PopulateAdditionalObjectSpaces(Application);  // lets lookups see secured spaces
        var dto = os.CreateObject<TSearchDTO>();
        var detail = Application.CreateDetailView(os, dto);
        detail.ViewEditMode = ViewEditMode.Edit;
        e.DialogController.SaveOnAccept = false;  // non-persistent — nothing to save
        e.View = detail;
    }

    private void Execute(object s, PopupWindowShowActionExecuteEventArgs e)
    {
        if (e.PopupWindowView is DetailView dv) dv.ObjectSpace.CommitChanges();
        var dto = e.PopupWindowViewCurrentObject as TSearchDTO;
        var criteria = CriteriaBuilder.BuildCriteria(dto, MaxActiveFilters);

        View.CollectionSource.Criteria.Remove(RequireSearchKey);
        if (criteria is not null)
            View.CollectionSource.Criteria[CriteriaKey] = criteria;
        else
            View.CollectionSource.Criteria.Remove(CriteriaKey);

        View.CollectionSource.Reload();
        View.Refresh();
    }
}
```

Per-entity controller:

```csharp
public class BatchSearchController : SearchControllerBase<Batch, BatchZoeker>
{
    protected override bool RequireSearchBeforeLoad => true;
}
```

## Key Patterns

### Require Search Before Load

For heavy list views (millions of rows), setting `Criteria["RequireSearchBeforeLoad"] = "1=0"` in `OnActivated` makes the grid render empty until the user runs a search. The action removes the key on execute.

This is more reliable than overriding `CollectionSource.AllowLoadCollection` — that approach fights XAF's lifecycle.

### Multiple Named Criteria

Use the `Criteria` dictionary, not a single string. Different controllers can stack filters without overwriting each other:

```csharp
View.CollectionSource.Criteria["RuntimeAdvancedSearch"] = userCriteria;
View.CollectionSource.Criteria["RequireSearchBeforeLoad"] = CriteriaOperator.Parse("1=0");
// XAF ANDs all entries automatically
```

### Wildcard + Exact Match in CriteriaBuilder

`CriteriaBuilder` (your helper) inspects DTO properties:

- String without `[UseExactMatch]` → translate `*` to `%`, `?` to `_`, wrap in `LIKE`.
- String with `[UseExactMatch]` → `=` comparison (status codes, enums).
- `DateTime? From` + `DateTime? To` pair → range; either side null = open-ended.
- All non-null DTO values are AND-ed.

### PopulateAdditionalObjectSpaces

`os.PopulateAdditionalObjectSpaces(Application)` is required so that lookup editors in the popup (e.g. an "Owner" dropdown that queries `ApplicationUser`) see the application's secured object spaces. Without this, lookups in the popup return empty.

### SaveOnAccept = false

The popup view is non-persistent. Setting `SaveOnAccept = false` prevents XAF from calling `CommitChanges` on a NonPersistentObjectSpace, which is a no-op but logs noise. The explicit `dv.ObjectSpace.CommitChanges()` in the execute handler is what flushes pending edits from the editor into the DTO instance.

## Generation Workflow (Out of Scope)

The code generator that turns `SearchConfiguration` rows into `BatchZoeker.cs` + `BatchSearchController.cs` is project-specific. Typical layout:

```
Search/
├── SearchObjects/      # generated DTOs (e.g. BatchZoeker.cs)
├── SearchControllers/  # generated controllers (e.g. BatchSearchController.cs)
├── Controllers/        # SearchControllerBase, SaveToProjectController
├── Services/           # SearchDtoCompiler (generates source strings)
└── Attributes/         # UseExactMatch, CollectionPath, EntityPath
```

Generated files are committed to source control. The "Save to Project" action writes them to disk via `File.WriteAllText`. Next build picks them up.

**Do not** use `Microsoft.CodeAnalysis.CSharp` for runtime compilation here — only for string-templating the generated source. Runtime `CSharpCompilation.Emit` + `AssemblyLoadContext` is exactly the failure mode that breaks `AddSecuredEFCore`.
