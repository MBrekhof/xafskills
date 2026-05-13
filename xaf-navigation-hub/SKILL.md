---
name: xaf-navigation-hub
description: >
  Replace the default DevExpress XAF sidebar navigation with a card-based DashboardView launchpad
  ("Navigation Hub") — categorized button tiles defined in the Application Model, filtered by per-user
  role permissions, with optional per-user pinned favorites. Use when XAF's default sidebar is
  overwhelming for users with many roles, or when you want a role-targeted "what should I do today"
  landing page. Covers IModelNavigationHub model extension, NavigationHubController.GetHubData()
  filtered by ShowNavigationItemController permissions, UserHubPreference entity, ImageLoader for
  tile icons, and the DashboardView-as-StartupNavigationItem wiring. Triggers: XAF navigation hub,
  XAF launchpad, card-based navigation XAF, IModelNavigationHubExtension, ShowNavigationItemController,
  UserHubPreference, replace XAF sidebar.
---

# XAF Card-Based Navigation Hub (Launchpad)

## Why Replace the Sidebar

XAF's default sidebar shows every accessible navigation item in a flat or shallowly grouped list. Users with 6+ roles see 30+ items and complain that they can't find their daily workflow. The Navigation Hub is a `DashboardView` set as `StartupNavigationItem` that renders categorized button tiles — admins curate the layout in the Application Model, security filters out items the user can't access, and users can pin favorites.

It composes with the existing navigation system; the sidebar still works, and clicking a tile uses the standard `ShowNavigationItemAction.DoExecute` pipeline so target views open the same way they always have.

## Model Extension

Define the hub structure as model interfaces so the layout lives in `Model.DesignedDiffs.xafml` (admin-editable) rather than in code:

```csharp
public interface IModelNavigationHubExtension : IModelNode
{
    IModelNavigationHub NavigationHub { get; }
}

public interface IModelNavigationHub : IModelNode, IModelList<IModelHubCategory> { }

[KeyProperty(nameof(Id))]
public interface IModelHubCategory : IModelNode
{
    string Id { get; set; }

    [Localizable(true)]
    string Caption { get; set; }

    int SortOrder { get; set; }

    IModelHubButtons Buttons { get; }
}

public interface IModelHubButtons : IModelNode, IModelList<IModelHubButton> { }

[KeyProperty(nameof(Id))]
public interface IModelHubButton : IModelNode
{
    string Id { get; set; }

    [Localizable(true)]
    string Caption { get; set; }

    string ImageName { get; set; }

    string NavigationItemId { get; set; }  // XAF navigation item ID path or ViewId

    string Color { get; set; }              // optional tile background tint

    int SortOrder { get; set; }

    string ExternalUrl { get; set; }        // alternative to NavigationItemId
}
```

Register the extension in your module's `ExtendModelInterfaces`:

```csharp
public override void ExtendModelInterfaces(ModelInterfaceExtenders extenders)
{
    base.ExtendModelInterfaces(extenders);
    extenders.Add<IModelApplication, IModelNavigationHubExtension>();
}
```

Admins now edit `Model > NavigationHub > Categories > Buttons` in the Model Editor and the xafml diff persists.

## NavigationHubController

A `WindowController` (not `ObjectViewController`) so it survives view switches and provides hub data to the platform-specific rendering component.

```csharp
public class NavigationHubController : WindowController
{
    public NavigationHubController()
    {
        TargetWindowType = WindowType.Main;
    }

    public List<HubCategoryData> GetHubData()
    {
        var model = (IModelNavigationHubExtension)Application.Model;
        var hubModel = model.NavigationHub;
        if (hubModel == null) return new();

        // Build the set of nav-item IDs the current user is allowed to see.
        var showNavController = Frame.GetController<ShowNavigationItemController>();
        var navAction = showNavController?.ShowNavigationItemAction;
        var permitted = new HashSet<string>();
        if (navAction != null) CollectPermittedItems(navAction.Items, permitted);

        var categories = new List<HubCategoryData>();
        foreach (IModelHubCategory cat in hubModel.OrderBy(c => c.SortOrder))
        {
            var buttons = new List<HubButtonData>();
            foreach (IModelHubButton btn in cat.Buttons.OrderBy(b => b.SortOrder))
            {
                var isExternal = !string.IsNullOrEmpty(btn.ExternalUrl);
                if (!isExternal
                    && !string.IsNullOrEmpty(btn.NavigationItemId)
                    && !permitted.Contains(btn.NavigationItemId))
                    continue;  // hide buttons the user lacks permission for

                buttons.Add(new HubButtonData
                {
                    Id = btn.Id,
                    Caption = btn.Caption,
                    ImageName = btn.ImageName,
                    ImageUrl = ResolveImageUrl(btn.ImageName),
                    NavigationItemId = btn.NavigationItemId,
                    Color = btn.Color,
                    ExternalUrl = btn.ExternalUrl ?? string.Empty
                });
            }
            if (buttons.Count > 0)
                categories.Add(new HubCategoryData
                {
                    Id = cat.Id,
                    Caption = cat.Caption,
                    Buttons = buttons
                });
        }
        return categories;
    }

    private void CollectPermittedItems(ChoiceActionItemCollection items, HashSet<string> ids)
    {
        foreach (ChoiceActionItem item in items)
        {
            if (item.Enabled && item.Active && item.Data is ViewShortcut shortcut)
            {
                ids.Add(item.GetIdPath());
                // Also add ViewId so buttons can target by either ID path or ViewId
                if (!string.IsNullOrEmpty(shortcut.ViewId)) ids.Add(shortcut.ViewId);
            }
            if (item.Items.Count > 0) CollectPermittedItems(item.Items, ids);
        }
    }

    public void NavigateToItem(string navigationItemId)
    {
        var showNavController = Frame.GetController<ShowNavigationItemController>();
        var navAction = showNavController?.ShowNavigationItemAction;
        if (navAction == null) return;

        var item = FindItemById(navAction.Items, navigationItemId);
        if (item != null && item.Enabled && item.Active)
            navAction.DoExecute(item);
    }

    private ChoiceActionItem FindItemById(ChoiceActionItemCollection items, string id)
    {
        foreach (var item in items)
        {
            if (item.GetIdPath() == id || (item.Data is ViewShortcut vs && vs.ViewId == id))
                return item;
            if (item.Items.Count > 0)
            {
                var found = FindItemById(item.Items, id);
                if (found != null) return found;
            }
        }
        return null;
    }
}

public class HubCategoryData
{
    public string Id { get; set; } = "";
    public string Caption { get; set; } = "";
    public List<HubButtonData> Buttons { get; set; } = new();
}

public class HubButtonData
{
    public string Id { get; set; } = "";
    public string Caption { get; set; } = "";
    public string ImageName { get; set; } = "";
    public string ImageUrl { get; set; } = "";
    public string NavigationItemId { get; set; } = "";
    public string Color { get; set; } = "";
    public string ExternalUrl { get; set; } = "";
}
```

Critical pieces:

- **Permission filtering via `ShowNavigationItemController.ShowNavigationItemAction.Items`** — this is the same item list XAF builds for the sidebar, already filtered by role permissions. Reusing it avoids re-implementing the permission check and guarantees the hub stays in sync with sidebar visibility.
- **Both `IdPath` and `ViewId` are tracked** so admins can target buttons either way without surprises. `Customer_NavGroup/Customer_ListView` (path) and `Customer_ListView` (view ID) both match.
- **`NavigateToItem` delegates to `DoExecute`** rather than calling `ShowViewStrategy` directly — this keeps any pre-navigation logic (security checks, criteria builders) the existing controllers attach to the action.
- **Hide empty categories** — `if (buttons.Count > 0)` so a category nobody can access doesn't render as an empty card.

## Per-User Pinned Favorites

```csharp
[DefaultClassOptions]
[VisibleInReports(false)]
public class UserHubPreference
{
    [Key] public virtual Guid Id { get; set; } = Guid.NewGuid();
    public virtual Guid UserId { get; set; }
    [MaxLength(256)] public virtual string NavigationItemId { get; set; } = string.Empty;
    public virtual int SortOrder { get; set; }
}
```

Read/write in the controller:

```csharp
public List<string> GetPinnedItemIds()
{
    if (SecuritySystem.CurrentUserId is not Guid userId) return new();
    using var os = Application.CreateObjectSpace(typeof(UserHubPreference));
    return os.GetObjects<UserHubPreference>(CriteriaOperator.Parse("UserId = ?", userId))
             .OrderBy(p => p.SortOrder)
             .Select(p => p.NavigationItemId)
             .ToList();
}

public void SetPinnedItems(List<string> ids)
{
    if (ids == null || SecuritySystem.CurrentUserId is not Guid userId) return;
    using var os = Application.CreateObjectSpace(typeof(UserHubPreference));
    var existing = os.GetObjects<UserHubPreference>(CriteriaOperator.Parse("UserId = ?", userId)).ToList();
    foreach (var e in existing) os.Delete(e);
    for (int i = 0; i < ids.Count; i++)
    {
        var pref = os.CreateObject<UserHubPreference>();
        pref.UserId = userId;
        pref.NavigationItemId = ids[i];
        pref.SortOrder = i;
    }
    os.CommitChanges();
}
```

Delete-and-recreate on save is simpler than diffing the collection — there are typically <20 pinned items per user.

## Image Resolution

XAF's `ImageLoader` returns image bytes or a URL depending on the source. The hub component needs a URL string for the `<img src>`, so embed bytes as a data URL when no URL is available:

```csharp
private static string ResolveImageUrl(string imageName)
{
    if (string.IsNullOrEmpty(imageName)) return "";
    var info = ImageLoader.Instance.GetLargeImageInfo(imageName);
    if (info.IsEmpty) info = ImageLoader.Instance.GetImageInfo(imageName);
    if (info.IsEmpty) return "";
    if (!info.IsUrlEmpty) return info.ImageUrl;
    if (info.ImageBytes is { Length: > 0 } bytes)
    {
        var mime = info.IsSvgImage ? "image/svg+xml" : "image/png";
        return $"data:{mime};base64,{Convert.ToBase64String(bytes)}";
    }
    return "";
}
```

Try `GetLargeImageInfo` first — hub tiles are 64-128px so the 32px versions look pixelated.

## Wiring the DashboardView as Startup

Create a `DashboardView` in the Application Model containing one `DashboardItem` that hosts the Blazor hub component (or WinForms control). In `Model > NavigationItems`:

- Add a `NavigationItem` whose `View` points at the new `DashboardView`.
- Set `Model > Options > StartupNavigationItem` to that NavigationItem's ID.

After login, the user lands on the hub instead of the previous startup view.

## Platform-Specific Rendering

The controller is in the shared Module; the actual rendering is platform-specific:

- **Blazor**: a Razor component that calls `ControllersManager.GetController<NavigationHubController>().GetHubData()`, renders the categories+tiles, and wires `@onclick="() => Controller.NavigateToItem(btn.NavigationItemId)"`. Use HTML5 drag-and-drop on the favorites strip for reordering.
- **WinForms**: an owner-drawn GDI+ control that paints the same data.

Both call the same controller methods, so security filtering and pin storage stay platform-agnostic.

## Gotchas

- **Don't pre-build hub data in the constructor** — `Frame` isn't assigned yet. Call `GetHubData()` from the rendering component (Razor `OnInitialized` or WinForms `OnLoad`).
- **Categories with zero permitted buttons must be hidden** — `if (buttons.Count > 0) categories.Add(...)`. Empty category cards confuse users.
- **External URL buttons skip the permission check** — by design, but make sure `NavigationItemId` is empty when `ExternalUrl` is set, or the filter will hide your external-link tile.
- **`StartupNavigationItem` is per-application, not per-role** — if different roles need different landing pages, override `IModelOptions.StartupNavigationItem` in a controller after login (read role, set startup, refresh).
- **Image bytes inflate the page payload** — for large hubs, prefer image files served from `wwwroot` over base64 data URLs.
