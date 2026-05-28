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
