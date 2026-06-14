---
name: xaf-easytest-authoring
description: >
  Author DevExpress EasyTest functional tests for XAF apps that drive the real WinForms and/or
  Blazor UI from one test body. Use when writing EasyTest functional tests, wiring an XAF project
  for EasyTest (EasyTest build config, remoting registration, auto-create+seed test DB, Blazor
  EasyTest adapter + browser driver), deriving test steps from entities/ViewControllers, or
  troubleshooting EasyTest failures (locale-bound dates, read-only nested grids, grid locator
  re-resolution, Blazor missing Close action, browser driver not found). Triggers: EasyTest, XAF
  functional test, EasyTestRemotingRegistration, IApplicationContext, GetGrid/FillForm/GetAction,
  GetValidationMessages, BlazorAdapter EasyTest, msedgedriver not found, XAF UI automation.
---

# Authoring XAF EasyTest Functional Tests

EasyTest drives the **real** XAF application — launches the actual host, logs in, clicks real
actions. Its API is **semantic**: you address elements by their *caption / column name / nav path*,
not by pixel or automation-id. That is exactly why it pairs well with an AI author — captions are
**derivable from the entity and controller source**, so tests are grounded in code, not guessed.

One test body can target **both** hosts via `[Theory]` + `[InlineData]`. EasyTest is often the only
practical automated-UI path for **WinForms**; the same body also drives **Blazor** (needs a browser
driver). Keep a test cross-platform when the flow is identical; make it single-platform (with a
comment saying why) when the UI interaction genuinely differs — nested-grid inline editing is the
usual culprit.

> **Verify DevExpress behavior through the dxdocs MCP** (`devexpress_docs_search` /
> `devexpress_docs_get_content`) before relying on any EasyTest API, attribute, or XAF behavior —
> method signatures, validation/edit-mode behavior, platform differences. If dxdocs has no answer,
> say so and label it an unverified assumption rather than guessing.

> **A project that already uses EasyTest usually ships its own `docs/EASYTEST-AUTHORING.md` (or
> equivalent).** Read it first — it holds the repo-specific facts this skill can't: actual nav
> names, field captions, seeded rows, connection wiring. This skill is the generic method + API +
> gotchas that apply to any XAF EasyTest suite.

---

## 1. Derive the test from the source (don't guess)

### Entities → captions, columns, nav items

Read each business object in the Module's `BusinessObjects/`:

| You need… | Read from the entity | Becomes in the test |
|---|---|---|
| Navigation item name | `[DefaultClassOptions]` present → object is in the nav | `app.Navigate("Customer")` |
| List grid column captions | property names (XAF humanizes `OrderNumber` → "Order Number") | `GetGrid().RowExists(("Order Number","SO-1001"))` |
| Detail form field captions | property names; `[DisplayName("…")]` overrides | `FillForm(("Order Number","SO-9001"))` |
| Lookup value to type | the related object's `[DefaultProperty]` | `FillForm(("Customer","Contoso Ltd"))` |
| What rows already exist | the seeding `Updater` (`ModuleUpdater.UpdateDatabaseAfterUpdateSchema`) | assert against known seed |
| Inline-editable nested grid | `[DefaultListViewOptions(true, …)]` on the child | `GetGrid("Order Lines").InlineNew()` |

**Humanization rule:** PascalCase → space-separated Title Case (`UnitPrice` → "Unit Price"). A
`[DisplayName]` / `[DefaultProperty]` attribute wins over the humanized name — check for those first.

### ViewControllers → actions and guards to assert

A controller's **actions** and **enable/visibility guards** are the highest-value things to test —
they're custom logic, not framework behavior. From a controller you can read the whole test without
running anything:

```csharp
markShipped = new SimpleAction(this, "MarkShipped", PredefinedCategory.RecordEdit) {
    Caption = "Mark Shipped",                                 // ← address by this caption
    SelectionDependencyType = SelectionDependencyType.RequireSingleObject
};
markShipped.Enabled["ConfirmedOnly"] =                        // ← the guard to assert
    View.CurrentObject is Order o && o.Status == OrderStatus.Confirmed;
markShipped.Execute += ...;                                   // ← effect: Confirmed → Shipped
```

→ caption `"Mark Shipped"`; guard ⇒ select a seeded **Confirmed** order; effect ⇒ assert the
`Status` cell flips to `"Shipped"`.

---

## 2. Test skeleton

`StartSeededApp` (a fixture helper) does drop-DB → launch host → login. Carry both `[InlineData]`s
when the flow is cross-platform; drop one and comment why when it isn't.

```csharp
[Theory]
[InlineData(WinAppName)]
[InlineData(BlazorAppName)]            // drop + comment when the flow is Win-only (see Gotcha 10)
public void MyFlow(string applicationName)
{
    var app = StartSeededApp(applicationName);   // fresh seeded DB + logged in
    app.Navigate("Customer");                    // nav item caption (entity with [DefaultClassOptions])
    // ... drive + assert ...
}
```

---

## 3. EasyTest API cheat-sheet

`app` is an `IApplicationContext`.

**Forms & actions**
```csharp
app.GetForm().FillForm(("User Name","Admin"), ("City","Berlin"));   // fields by caption
app.GetAction("Log In").Execute();                                  // toolbar/action by caption
app.GetAction("Save").Execute(); app.GetAction("New").Execute(); app.GetAction("Close").Execute();
app.Navigate("Order");                                              // nav; dotted path e.g. "Reports.Reports"
```

**Grid — read**
```csharp
app.GetGrid().GetRowCount();
app.GetGrid().RowExists(("Name","Contoso Ltd"));
int? i = app.GetGrid().GetRowIndex(("Order Number","SO-1001"));     // nullable!
app.GetGrid().GetRow(i.Value, "Order Number","Status");            // string[] of those columns
app.GetGrid().GetRows("Name","City");                              // all rows
```

**Grid — select / inline edit**
```csharp
app.GetGrid().SelectRows("Order Number","SO-1001");                // then run a ListView action
app.GetGrid().ProcessRow(("Order Number","SO-1001"));             // open the row's DetailView
var lines = app.GetGrid("Order Lines");                           // nested grid by caption
lines.InlineNew(); lines.FillRow(("Product Name","X"),("Quantity","3")); lines.InlineUpdate();
```

**Validation** (after a Save that breaks a rule)
```csharp
app.GetAction("Save").Execute();                                  // triggers Save-context rules
var msgs = app.GetValidation().GetValidationMessages();           // string[] of displayed messages
Assert.Contains(msgs, m => m.Contains("open"));
// app.GetValidation().GetValidationHeader();                     // Win-only popup header
```
Test validation by reading the **displayed messages** — don't assume the Save threw. Prefer
`Contains` over exact equality (messages get prefixed/formatted per platform).

---

## 4. Gotchas (each one earned by a failing run)

1. **Dates are locale-bound.** A date editor expects the **host's short-date culture** (e.g. `nl-NL`
   = `dd-MM-yyyy`). `"6/13/2026"` throws `"…doesn't correspond to the 'd' target input format"`.
   Omit non-required dates, or format to the host culture.
2. **Nested grids are read-only by default.** `InlineNew()` fails with
   `Cannot get the ActiveEditor … (-1, …)` unless the child entity has
   `[DefaultListViewOptions(true, NewItemRowPosition.Top)]` (or the list view's `AllowEdit` is set).
3. **Assert nested-grid rows *before* `Save`.** After a Save the nested-grid locator re-resolves to a
   toolbar button (`IGridBase … not supported … BarButtonItemLink`). `RowExists` while the grid is
   the active control, *then* Save.
4. **Nested in-line edits persist only on the *root* Save.** `InlineUpdate()` commits the row into
   the parent's in-memory collection; the DB write happens when you Save the parent detail.
5. **`GetRowIndex` returns `int?`** — null when the row is absent. Null-check before `.Value`.
6. **Win is not headless.** It opens the real app window → needs an **interactive Windows desktop
   session**. No desktop → nothing runs (rules out naive headless CI agents).
7. **Build config matters.** The fixture launches the **EasyTest-config** builds of the hosts. Forget
   `-c EasyTest` and you run a stale/Debug build (wrong DB connection, no remoting/adapter) — tests
   hang or hit the wrong database.
8. **No `Close` action on Blazor.** After a Save, Win has a `Close` action; Blazor doesn't, so
   `GetAction("Close")` returns null → NRE. Don't `Close` to leave a saved detail — just `Navigate`
   away (clean on both once there are no unsaved changes).
9. **Blazor driver isn't auto-managed.** The `EasyTest.BlazorAdapter` throws `Browser driver is not
   found` unless a version-matched `msedgedriver.exe` (matching installed Edge) is on PATH or pointed
   to by `webDriverPath`. Raw Selenium auto-downloads; the DevExpress adapter does not. Refresh the
   driver when Edge updates. `runHeadless: true` works for CI but still needs the driver.
10. **Some flows are platform-specific.** Nested-grid in-place editing is the main one — Win uses a
    New Item Row; Blazor uses a different `InlineEditMode`/edit-form. Such a test is single-platform
    until you write the other host's line-entry steps.

---

## 5. One-time project wiring (replicate per project)

To make an XAF project EasyTest-able. All host changes go under `#if EASYTEST` so normal builds are
untouched; add an `EasyTest` solution configuration (`Debug;Release;EasyTest`).

| Requirement | Where | Notes |
|---|---|---|
| `EasyTest` solution configuration | `Configurations` in host + Module csproj | Three configs incl. `EasyTest` |
| EasyTest remoting registration | host `Program.cs` | `EasyTestRemotingRegistration.Register()` under `#if EASYTEST` |
| Auto-create + seed DB under test | host `WinApplication.cs` / `BlazorApplication.cs` | `DatabaseVersionMismatch` → `e.Updater.Update()` under `#if EASYTEST` |
| Dedicated test DB connection | host `App.config` / `appsettings.json` | switch to an `EasyTestConnectionString` under `EASYTEST` — separate catalog from the real app DB |
| Idempotent seed | Module `DatabaseUpdate/Updater.cs` | re-runs on every drop+recreate; tests assert against it |
| Test fixture | E2E test project | drops the test catalog before each test; registers the hosts; non-parallel |
| **Blazor** EasyTest adapter | Blazor host csproj | `DevExpress.ExpressApp.EasyTest.BlazorAdapter` — referenced **only in the `EasyTest` config** |
| **Blazor** browser driver | `tools/webdriver/msedgedriver.exe` (git-ignored) | version-matched to Edge; passed via `webDriverPath` in `BlazorApplicationOptions` |

The fixture **drops a dedicated test catalog before each test**; the host recreates the schema and
re-runs the seed `Updater` on launch — so every test starts from a known, freshly-seeded state, and
the real app catalog is never touched.

```sh
# Build the hosts in the EasyTest config (the fixture launches these), then run:
dotnet build <Host>.csproj -c EasyTest
dotnet test  <E2E.Tests>.csproj --no-build --filter "FullyQualifiedName~MarkOrderShipped"
```

---

## 6. Checklist for adding a test

1. **Identify the flow** — from the request or a controller's logic.
2. **Read the entity/entities** → exact nav name, field captions, column names, lookup display
   values. Humanize, or read `[DisplayName]` / `[DefaultProperty]`. Don't guess captions.
3. **Read any ViewController** in play → action captions + enable/visibility guards to assert.
4. **Check the seed** for rows you can rely on, or create what you need.
5. **Verify any DevExpress assumption via dxdocs** — EasyTest signatures, validation/edit-mode
   behavior, attribute constructors, platform differences. Don't rely on memory for DX APIs.
6. **Write the test** from the skeleton (§2) using the cheat-sheet (§3); apply the gotchas (§4).
7. **Build `-c EasyTest`, run with a `--filter`**, iterate. On failure, read the EasyTest exception —
   it names the control/field/format precisely (that's how every gotcha above was found).
8. If a flow needs UI config that doesn't exist yet (e.g. an editable nested grid), prefer a **code
   attribute** (`[DefaultListViewOptions]`) over hand-editing `.xafml`.
9. **Test both sides of an invariant** — a `RuleFromBoolProperty` (true = valid) → one test that
   drives the object invalid and reads `GetValidationMessages()`, one that drives it valid and
   confirms the Save persisted (`RowExists`).
