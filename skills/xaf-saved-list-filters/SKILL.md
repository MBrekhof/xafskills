---
name: xaf-saved-list-filters
description: >
  Add or maintain saved filters for DevExpress XAF ListViews with EF Core. Use when users need
  to save, name, share, select, clear, or make a filter the default. Covers the persistent filter
  entity, current-user and tenant visibility, ViewId scoping, CriteriaOperator parsing, ListView
  actions, grid-editor fallbacks, safe criteria clearing, and stale-filter handling. Do not use
  for ordinary ad hoc CollectionSource criteria that are not saved for reuse. Triggers: XAF saved
  filters, FilteringCriteria, FilteringCriterion, CriteriaListViewController, saved ListView filter.
---

# XAF Saved List Filters

Use this skill to add saved filters to an XAF application with EF Core. A saved filter is a persistent record that stores a Criteria string for later use on a ListView. The controller presents the records as actions and applies the selected criterion through the view's `CollectionSource`.

The reference design is [saved-list-filters-design.md](../../docs/saved-list-filters-design.md). The comparison with Fleetman, HIS, PathQ, and DataDrive is in [saved-list-filters-comparison.md](../../docs/saved-list-filters-comparison.md). Read the parts relevant to the current task; first inspect the target application because user, tenant, ORM, security, and editor choices differ.

## Before changing code

Inspect the target project and identify:

- XAF and EF Core versions, Blazor or WinForms hosts, and the module that owns shared ListView behavior.
- Existing saved-filter entities, controllers, `CollectionSource.Criteria` keys, and editor APIs.
- The user type and security model, including who can create public filters.
- Whether tenants use separate databases, schemas, or rows in a shared database.
- How the app scopes `IObjectSpace` and applies tenant isolation.
- Which ListViews and editor types must support reading and applying filters.

Check current DevExpress documentation for APIs whose signatures or platform behavior matter. Don't copy an XPO implementation into EF Core or assume Blazor and WinForms expose filter state identically.

## Data model

Follow the application's EF Core base-entity and naming conventions. A filter entity typically needs:

- `Name` and `Criterion`.
- Persisted `ObjectTypeFullName` and a non-persistent `ObjectType` for the criteria editor.
- `ViewId` when the same object type has multiple semantically different ListViews.
- An owner reference and an `AllowPublic` flag when private and shared filters are required.
- A default flag only if the product needs automatic filter selection.
- Tenant scope only when the persistence boundary does not already isolate tenants.

Use `[CriteriaOptions(nameof(ObjectType))]` on the criteria property. Resolve stored type names through the target app's XAF type metadata where available, such as `XafTypesInfo.Instance.FindTypeInfo(...)`. Handle unresolved names as stale filters; don't let them break view activation.

Register the entity in the EF Core context or the project's established type-registration mechanism. Add a migration and XAF type permissions. Match the permission granularity to the app's security setup.

## Query and access rules

Load filter records through the current secured `IObjectSpace`. Avoid a process-wide static list of XAF objects: Blazor Server serves multiple users in one process, and a global cache can leak tenant data or retain objects from a disposed ObjectSpace.

Filter available records by all applicable scopes:

1. Tenant or isolated database context.
2. Compatible object type. A base-type filter may match derived types only if that behavior is intended.
3. Matching `ViewId`, or an explicitly supported type-wide filter.
4. Public visibility or ownership by the current user.
5. XAF read permission for the saved-filter entity.

If users share filters by role, add an explicit role relationship and check it in the same access path. Do not infer role access from `AllowPublic`.

For applications with a `Default` role, use this baseline unless the product specifies otherwise: allow users to create filters, read their own and shared filters, and write their own filters; deny deletion. Keep these permissions and row scopes explicit on the `Default` role. Do not change other roles as part of this baseline. A different role may grant broader access under the application's role-merging policy; treat that as separate configuration unless the task asks for an application-wide policy.

The saved criterion does not grant access to business records. Keep XAF security on the target ListView and use a secured ObjectSpace. A public filter means that eligible users can reuse the criterion; it must not bypass row or object permissions.

Default filters need deterministic scope. Prefer a user's default over a public default for the same tenant, type, and view. Enforce at most one default per scope when saving. If legacy data contains duplicates, report the conflict and avoid selecting one arbitrarily.

## ListView controller

Use a `ViewController<ListView>` for actions shared by supported list views. Restrict the actions to root views when nested lists are not part of the feature. Keep the action IDs stable and set captions explicitly.

Keep persistence, visibility, validation, and default-selection rules in a small service when more than one controller or entry point needs them. For a narrow single-view feature, follow the project's simpler established pattern without creating an unnecessary service layer.

The controller should:

1. Query available filters when the view is ready.
2. Add filter choices and an “All” choice to a `SingleChoiceAction`.
3. Select the applicable default only after loading accessible choices.
4. Apply a selected criterion and synchronize the editor's visible filter where supported.
5. Open a modal `DetailView` to create or edit a filter.
6. Refresh choices only after a successful commit.
7. Clear only criteria whose ownership is known.

Subscribe to events in the matching activation lifecycle and unsubscribe in `OnDeactivated`. Track modal ObjectSpaces and dialog handlers so they do not outlive the view.

## Reading and applying criteria

When saving the current filter, read it from the most reliable supported source:

1. `ISupportFilter.Filter`, when the active editor exposes it.
2. The platform-specific grid API, such as the current DxGrid adapter's `GetFilterCriteria()`.
3. The saved-filter controller's own `CollectionSource.Criteria` entry, if the application supports that path.

Serialize a `CriteriaOperator` with the DevExpress API. Don't build Criteria syntax by concatenating user values into a string.

When applying a saved filter:

1. Recheck that the user can read and use the saved record.
2. Resolve its object type and parse the stored text with the current `IObjectSpace`.
3. Put the result under a stable key owned by this controller, for example `nameof(SavedListFilterListViewController)`.
4. Update `ISupportFilter.Filter` only when the editor supports it.
5. Refresh the ListView.

Validate on save and again on use. A property or relationship can be renamed after the filter was created. If parsing fails, skip that choice or return to “All,” log the filter identifier and error, and show a useful message when the user selected it. One stale filter must not prevent the whole ListView from opening.

## Saving and refreshing

Create the filter in an ObjectSpace appropriate for its secured entity. Initialize owner, tenant, type, `ViewId`, and private visibility in code; do not rely on the user to fill security-scope fields correctly.

Use a modal DetailView for user-editable fields such as name, criterion, visibility, and default state. Before commit, verify ownership, permission to publish, tenant scope, and default uniqueness. Commit first; then reload or refresh the action items. Don't update a shared cache in an entity `OnSaving` callback before the transaction has committed.

If the user can manage filters in a separate list, scope that list to the same tenant, type, view, and ownership rules. Apply the same edit checks there. If deletion is a product requirement, enforce its permission there too; hiding an action does not secure the entity.

## Clearing filters safely

`CollectionSource.Criteria` can contain entries from saved filters, full-text search, column filters, security controllers, and application features. Use a dedicated key for the saved filter and remove only that key when replacing or clearing it.

Before clearing any additional key, find its owner and contract in the target project. Preserve tenant, permission, and safety criteria. Don't assume `View.Model.Filter = string.Empty` clears runtime criteria or the editor's displayed filter; synchronize each layer that this UI actually uses.

If the product wants a global “clear user filters” action, make its list of removable keys explicit and keep protected criteria out of that list.

## Keep the first version focused

Do not add these without a product requirement:

- Persisted DxGrid layout, sorting, grouping, or paging state.
- Role-based sharing.
- Static or distributed cache.
- Automatic rewriting of saved criteria after a model refactor.
- Creating conditional-appearance rules from filters.

These are separate features with their own storage, access, and compatibility concerns.

## Completion checklist

- A user can save, select, rename, and clear a filter on each required platform.
- Private and public visibility behave correctly for different users.
- The `Default` role can create and read its permitted filters, can write owned filters, and cannot delete filters when this baseline applies.
- Tenant boundaries are enforced by the data access path.
- Type and `ViewId` scope match the product's intended behavior.
- Default selection is deterministic and duplicate defaults are handled.
- Invalid or stale criteria do not break view activation.
- Clearing a saved filter preserves security and unrelated controller criteria.
- Filter choices refresh after commit without restarting the application.
- New entity permissions and database migration follow project conventions.
