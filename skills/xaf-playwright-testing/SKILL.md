---
name: xaf-playwright-testing
description: >
  E2E test DevExpress XAF Blazor Server applications with Playwright. Use when setting up a
  Playwright test project for an XAF app, writing page objects for XAF Blazor pages, handling
  the XAF login flow, or troubleshooting flaky XAF E2E tests. Covers Microsoft.Playwright.NUnit
  + PageTest base class, robust selectors for DevExpress Blazor components, screenshot-on-failure,
  per-test authentication, and the localization-resilient locator pattern. Triggers: XAF Playwright,
  XAF E2E testing, XAF Blazor automation, DevExtreme Blazor selector, PageTest XAF, AuthenticatedTestBase.
---

# XAF Blazor E2E Testing with Playwright

## Project Layout

```
YourApp.Playwright/
├── Configuration/TestSettings.cs       # appsettings.test.json binding
├── Fixtures/AuthenticatedTestBase.cs   # login + screenshot fixture
├── Pages/                              # Page Objects (LoginPage, MainPage, ...)
├── Tests/                              # NUnit test classes
├── appsettings.test.json
└── .runsettings
```

## Required Packages

```xml
<PackageReference Include="Microsoft.Playwright.NUnit" Version="1.58.0" />
<PackageReference Include="NUnit" Version="3.14.0" />
<PackageReference Include="NUnit3TestAdapter" Version="4.5.0" />
<PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
<PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="10.0.4" />
<PackageReference Include="Microsoft.Extensions.Configuration.Binder" Version="10.0.4" />
```

Copy `appsettings.test.json` and `.runsettings` to output:

```xml
<ItemGroup>
  <None Update="appsettings.test.json"><CopyToOutputDirectory>Always</CopyToOutputDirectory></None>
  <None Update=".runsettings"><CopyToOutputDirectory>Always</CopyToOutputDirectory></None>
</ItemGroup>
<ItemGroup>
  <Using Include="NUnit.Framework" />
</ItemGroup>
```

After first build, install browsers:

```powershell
powershell -File YourApp.Playwright/bin/Debug/net8.0/playwright.ps1 install chromium
```

## Authenticated Test Base

Inherit `PageTest` from `Microsoft.Playwright.NUnit` — it gives you `Page`, `Context`, and `Browser` per test automatically. Do **not** spin up your own playwright instance.

```csharp
public class AuthenticatedTestBase : PageTest
{
    protected TestSettings Settings => TestSettings.Instance;
    protected LoginPage LoginPage => new(Page);

    public override BrowserNewContextOptions ContextOptions() => new()
    {
        ViewportSize = new ViewportSize { Width = 1920, Height = 1080 },
        IgnoreHTTPSErrors = true,  // XAF dev cert
    };

    [SetUp]
    public async Task SetUpAuthentication()
    {
        Page.SetDefaultTimeout(Settings.DefaultTimeout);
        Page.SetDefaultNavigationTimeout(Settings.DefaultTimeout);

        await Page.GotoAsync(Settings.BaseUrl, new()
        {
            WaitUntil = WaitUntilState.NetworkIdle,  // XAF Blazor wait
            Timeout = Settings.DefaultTimeout
        });

        var login = new LoginPage(Page);
        if (await login.IsVisibleAsync())
            await login.LoginAsync(Settings.AdminUser, Settings.AdminPassword);
    }

    [TearDown]
    public async Task TearDownScreenshot()
    {
        if (TestContext.CurrentContext.Result.Outcome.Status == TestStatus.Failed
            && Settings.Screenshots.OnFailure)
        {
            await TakeScreenshotAsync();
        }
    }
}
```

`IgnoreHTTPSErrors = true` is mandatory if you hit XAF on the local dev cert.

## XAF Login Page Selectors

XAF Blazor renders login form elements with inconsistent IDs/classes depending on theme, locale, and DevExpress version. Use **multi-fallback locators** with `.First` so one broken selector doesn't fail the whole suite.

```csharp
public ILocator UserNameInput => _page.Locator(
    "input[name='UserName'], #UserName, input[placeholder*='user' i], " +
    "input[aria-label*='user' i], .dxbl-text-edit input[type='text']").First;

public ILocator LoginButton => _page.Locator(
    "button:has-text('Log In'), button:has-text('Aanmelden'), " +
    "input[type='submit'], button[type='submit']").First;
```

The `[placeholder*='user' i]` (case-insensitive contains) and multi-language button text (`Log In`, `Aanmelden`) make tests survive UI rebrands and localization changes.

## Page Object Pattern

One class per logical page. Hold the `IPage`, expose `ILocator` properties for elements, and add task-oriented `async` methods.

```csharp
public class LoginPage(IPage page)
{
    private readonly IPage _page = page;

    public ILocator UserNameInput => _page.Locator(...).First;
    public ILocator PasswordInput => _page.Locator(...).First;
    public ILocator LoginButton => _page.Locator(...).First;

    public async Task LoginAsync(string user, string pwd)
    {
        await UserNameInput.WaitForAsync(new() { State = WaitForSelectorState.Visible, Timeout = 15000 });
        await UserNameInput.FillAsync(user);
        await PasswordInput.FillAsync(pwd);
        await LoginButton.ClickAsync();
    }

    public async Task<bool> IsVisibleAsync(float timeout = 5000)
    {
        try
        {
            await UserNameInput.WaitForAsync(new() { State = WaitForSelectorState.Visible, Timeout = timeout });
            return true;
        }
        catch { return false; }
    }
}
```

`IsVisibleAsync` swallowing the timeout exception lets the fixture branch ("if login page visible, log in") without try/catch noise in callers.

## TestSettings Singleton

```csharp
public sealed class TestSettings
{
    private static readonly Lazy<TestSettings> _instance = new(() =>
    {
        var cfg = new ConfigurationBuilder()
            .SetBasePath(AppContext.BaseDirectory)
            .AddJsonFile("appsettings.test.json", optional: false, reloadOnChange: false)
            .Build();
        var s = new TestSettings();
        cfg.Bind(s);
        return s;
    });

    public static TestSettings Instance => _instance.Value;

    public string BaseUrl { get; set; } = "https://localhost:5003";
    public string AdminUser { get; set; } = "Admin";
    public string AdminPassword { get; set; } = "";
    public float DefaultTimeout { get; set; } = 30000;
    public ScreenshotSettings Screenshots { get; set; } = new();
}
```

`appsettings.test.json` lives alongside the test DLL and is **gitignored if it contains passwords**.

## Screenshot on Failure

```csharp
protected async Task TakeScreenshotAsync(string? suffix = null)
{
    var dir = Path.Combine(AppContext.BaseDirectory, Settings.Screenshots.Directory);
    Directory.CreateDirectory(dir);

    var name = TestContext.CurrentContext.Test.Name;
    var ts = DateTime.Now.ToString("yyyy-MM-dd_HH-mm-ss");
    var path = Path.Combine(dir, $"{name}_{ts}{(suffix is null ? "" : "_" + suffix)}.png");

    await Page.ScreenshotAsync(new() { Path = path, FullPage = true });
    TestContext.AddTestAttachment(path);
}
```

`TestContext.AddTestAttachment` makes the screenshot show up in the NUnit test result, which is critical in CI.

## Waiting for XAF Blazor

XAF Blazor renders SignalR-over-WebSocket. `WaitUntil = Load` returns *before* the circuit is ready and `NetworkIdle` is the only reliable signal for initial navigation.

For post-click waits, **never** use `Task.Delay` — use Playwright's web-first assertions:

```csharp
await Expect(NavigationHubPage.HubContainer).ToBeVisibleAsync();
await Expect(Page.Locator(".dxbl-grid-data-row")).Not.ToBeEmptyAsync();
```

`Expect` auto-retries until the timeout expires, which handles the circuit reconnect/rerender race.

## .runsettings — Headed Mode for Local Debugging

```xml
<RunSettings>
  <Playwright>
    <BrowserName>chromium</BrowserName>
    <LaunchOptions>
      <Headless>false</Headless>
      <SlowMo>250</SlowMo>
    </LaunchOptions>
  </Playwright>
</RunSettings>
```

Headless is default in CI. Locally, run with `dotnet test --settings .runsettings` to watch the browser.

## Running

```bash
# Headed local
dotnet test YourApp.Playwright --settings YourApp.Playwright/.runsettings

# Headless CI
dotnet test YourApp.Playwright
```

The XAF app **must already be running** at `BaseUrl` before the tests start. These tests do not host the app.

## Gotchas

- **`#if DEBUG` auth modes**: if your XAF app forces SSO in Release builds, your Playwright tests need either Test environment (see `xaf-environment-auth`) or a publish profile that disables SSO. Otherwise login fails silently.
- **PageTest vs Playwright**: do NOT use `[OneTimeSetUp]` to instantiate `IPlaywright` yourself — `PageTest` already manages it. You'll get "browser closed" errors if you mix.
- **NetworkIdle + Hangfire**: if your app embeds Hangfire and the dashboard polls in the background, `NetworkIdle` may never fire. Switch to `DOMContentLoaded` + an explicit `Expect(...).ToBeVisibleAsync()` for a known element.
- **Locator `.First`**: required when a selector matches multiple candidates (XAF wraps inputs in nested editors). Without `.First`, Playwright throws strict-mode violation.
- **Browsers**: `playwright install chromium` must be re-run after every Playwright NuGet upgrade. In CI, run it as a build step.
