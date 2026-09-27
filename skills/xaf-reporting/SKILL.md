---
name: xaf-reporting
description: >
  Implement DevExpress XAF ReportsV2 with custom parameter objects. Use when creating reports with
  ReportParametersObjectBase, registering predefined reports, or troubleshooting report parameter issues.
  Covers the critical Visible=false gotcha, GetCriteria() vs FilterString, PredefinedReportsUpdater,
  and the DomainComponent requirement. Triggers on XAF reporting work.
---

# XAF ReportsV2 & Parameter Objects

## Critical Gotcha: Parameter Visibility

XtraReport parameters with `Visible = true` cause the report viewer to show its own parameter panel, **completely bypassing** XAF's `ReportParametersObjectBase` detail view.

```csharp
var param = new Parameter
{
    Name = "CustomerName",
    Type = typeof(string),
    Visible = false  // CRITICAL: Must be false for ReportParametersObjectBase to work
};
```

Set `Visible = false` on ALL parameters in the XtraReport.

## GetCriteria() vs FilterString

`?paramName` substitution in XtraReport's `FilterString` does NOT work with `ReportParametersObjectBase`. The framework passes the entire parameter object as one hidden parameter, not individual values.

```csharp
// WRONG - FilterString substitution doesn't work
report.FilterString = "[Customer.Name] = ?CustomerName";

// CORRECT - Override GetCriteria()
public override CriteriaOperator GetCriteria()
{
    var criteria = new List<CriteriaOperator>();
    if (!string.IsNullOrEmpty(CustomerName))
        criteria.Add(CriteriaOperator.Parse("Customer.Name = ?", CustomerName));
    if (StartDate != default)
        criteria.Add(CriteriaOperator.Parse("OrderDate >= ?", StartDate));
    return CriteriaOperator.And(criteria);
}
```

`GetCriteria()` is called in both Blazor Server and WinForms via `ReportServiceController.OnHandleAccepted()`.

## Parameter Object Requirements

### [DomainComponent] is mandatory

```csharp
[DomainComponent]
public class OrdersReportParameters : ReportParametersObjectBase
{
    public OrdersReportParameters(IObjectSpaceCreator provider) : base(provider) { }

    public string? CustomerName { get; set; }
    public DateTime StartDate { get; set; } = DateTime.Today.AddMonths(-1);

    public override CriteriaOperator GetCriteria() { /* ... */ }
    public override SortProperty[] GetSorting() => Array.Empty<SortProperty>();
}
```

- Must have `[DomainComponent]` attribute
- Must have constructor accepting `IObjectSpaceCreator`
- XAF automatically shows a DetailView for the parameter object before opening the report

### Lookup parameters (business object references)

```csharp
// For lookup parameters, check null and use .ID
if (Customer is not null)
    criteria.Add(CriteriaOperator.Parse("Customer.ID = ?", Customer.ID));
```

Verified end-to-end on Blazor (2026-08-22, XafReportParametersObjects): a `Customer?` property on
a `[DomainComponent]` parameters object renders as a normal XAF lookup in the parameters dialog.
The XtraReport side can carry the same intent — `new Parameter { Type = typeof(Customer), Visible = false }`
is a legal custom-typed parameter (dxdocs XtraReports/9999) and survives REPX serialization
(Copy Predefined Report → `IReportStorage.LoadReport`), so tooling can detect lookups by walking
`Parameter.Type.BaseType` to `DevExpress.Persistent.BaseImpl.EF.BaseObject`.

### Range filters

```csharp
// Date ranges: inclusive start, exclusive end (include full day)
if (StartDate != default)
    criteria.Add(CriteriaOperator.Parse("OrderDate >= ?", StartDate.Date));
if (EndDate != default)
    criteria.Add(CriteriaOperator.Parse("OrderDate < ?", EndDate.Date.AddDays(1)));
```

## Registering Predefined Reports

In the Module class:

```csharp
public override IEnumerable<ModuleUpdater> GetModuleUpdaters(
    IObjectSpace objectSpace, Version versionFromDB)
{
    var updater = new DatabaseUpdate.Updater(objectSpace, versionFromDB);

    var reportsUpdater = new PredefinedReportsUpdater(Application, objectSpace, versionFromDB);
    reportsUpdater.AddPredefinedReport<OrdersReport>(
        "Orders Report",           // Display name
        typeof(Order),             // Data type
        typeof(OrdersReportParameters));  // Parameter object type (optional)

    return new ModuleUpdater[] { updater, reportsUpdater };
}
```

## Report Module Configuration

In Startup.cs builder:

```csharp
builder.Modules.AddReports(options => {
    options.EnableInplaceReports = true;
    options.ReportDataType = typeof(DevExpress.Persistent.BaseImpl.EF.ReportDataV2);
    options.ReportStoreMode = ReportStoreModes.XML;  // XML in database, not files
});
```

## Exposing Reports via Web API

Register `ReportDataV2` for OData access:

```csharp
webApiBuilder.ConfigureOptions(options => {
    options.BusinessObject<DevExpress.Persistent.BaseImpl.EF.ReportDataV2>();
});
```

Add `DbSet<ReportDataV2>` to the DbContext.

## Programmatic Report Creation

```csharp
public class OrdersReport : XtraReport
{
    public OrdersReport()
    {
        var param = new Parameter {
            Name = "StartDate",
            Type = typeof(DateTime),
            Visible = false  // Always false!
        };
        Parameters.Add(param);

        var dataSource = new CollectionDataSource { ObjectTypeName = typeof(Order).FullName };
        DataSource = dataSource;

        // Build bands, controls, etc.
    }
}
```

## Report Parameter Signature Hashing

For detecting when a report's parameters have changed from the generated parameter object, hash parameter signatures:

```csharp
var parts = report.Parameters.Cast<Parameter>()
    .Select(p => $"{p.Name}:{p.Type?.FullName}")
    .OrderBy(s => s);
var hash = SHA256(string.Join("|", parts));
```

Compare stored hash with current to set an `IsStale` flag for regeneration.

## Predefined Reports Own Their ParametersObjectType

`PredefinedReportsUpdater` reconciles its registrations on every database update
(via its `ReportDataComparer`). If code changes a predefined report's
`ParametersObjectType` to something other than what `AddPredefinedReport` declared,
the row is treated as orphaned: it disappears from the Reports list and a new
canonical row is created — you get duplicate report rows on every reconciliation.

Rules:
- Predefined reports: declare the parameters type in `AddPredefinedReport<T>(name, dataType, parametersType)`. Never reassign it at runtime.
- Dynamically assigning `ParametersObjectType` (e.g. to a generated class) is only safe for **user-created reports** (`PredefinedReportTypeName` is null/empty). Guard any updater logic with that check.
- To experiment with a predefined report, use the built-in **Copy Predefined Report** action to get a user-owned copy first.

## DomainComponent Classes Are Auto-Collected in Module Assemblies

Non-persistent `[DomainComponent]` classes declared inside an XAF module assembly are
collected automatically by `ModuleBase.GetDeclaredExportedTypes()` — no
`AdditionalExportedTypes.Add(...)` needed (verified on 25.2: a generated
`ReportParametersObjectBase` subclass worked with no explicit registration).
`AdditionalExportedTypes` is for types in *other* (non-module) assemblies.

## Template Gotcha: DB Schema Update Requires an Attached Debugger

The XAF solution template only auto-updates the database on version/schema mismatch when
`System.Diagnostics.Debugger.IsAttached` (i.e., F5 from Visual Studio). Running
`dotnet run` headless after a model change throws the DatabaseVersionMismatch error.
For dev workflows that run from the CLI, change the template handlers to update in
`#if DEBUG` regardless of debugger (BlazorApplication.cs / WinApplication.cs +
Win Startup.cs `DatabaseUpdateMode` block).
