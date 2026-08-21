---
name: add-playwright-tests-to-blazor
description: Add a Playwright (Microsoft.Playwright.Xunit) end-to-end test project to an existing ASP.NET Core Blazor Server (or similar) web app, including an auth bypass for Entra ID/OIDC-protected apps and a real running instance of the app for the browser to navigate to. Use when the user asks to add Playwright tests, browser tests, or end-to-end/UI tests to a Blazor or ASP.NET Core project.
---

# Add Playwright Tests to a Blazor / ASP.NET Core Project

Use this skill to scaffold a Playwright-based end-to-end test project against a real, running
instance of an existing ASP.NET Core app (Blazor Server in particular), including handling of
authentication and static asset gotchas discovered the hard way. Read this whole file before
starting — the "Problems encountered" section documents non-obvious failures that otherwise cost
a lot of iteration.

## Before starting — ask the user

1. **Check whether the solution has one or more Aspire AppHost projects** (a `*.AppHost.csproj`
   using the `Aspire.AppHost.Sdk`, typically alongside a `*.ServiceDefaults` project).
   - If **none** exist, use the process-launch variant ("Step 5 (process-launch variant)")
     without asking — there's nothing to choose between.
   - If **one or more** exist, ask the user how to host the app under test:
     - Use a specific AppHost project (list them by name if there's more than one) via the Aspire
       variant ("Step 5 (Aspire variant)") — this starts real container-backed dependencies (e.g.
       SQL Server) automatically, which is more realistic but requires Docker.
     - Use the plain process-launch variant instead (launch the web app's own `dotnet <dll>`
       directly, with its own LocalDB-based `appsettings.Testing.json`), even though an AppHost
       exists — simpler and faster to start, no Docker dependency.
   - Do not add Aspire to a solution solely for this skill — only offer the Aspire variant if an
     AppHost already exists.
2. **Auth handling.** If the app requires real login (Entra ID/OIDC, Identity, etc.), ask whether
   to add a test-only auth bypass (recommended — a fake `AuthenticationHandler` activated only in
   a `Testing` ASPNETCORE_ENVIRONMENT) or whether the tests should perform a real login.
3. **Which page(s)/flows** to cover first (start small: one list page with CRUD-ish interactions
   is a good default scope).
4. Confirm test project style: **xUnit + Microsoft.Playwright.Xunit** is the default for a C#/.NET
   solution — keep it consistent with the rest of the solution. Use NUnit instead only if the user
   asks for it or the solution's other test projects already use NUnit (its equivalent building
   blocks are noted alongside the xUnit code in Steps 4–7).

## Overview of the approach

Do **not** use `WebApplicationFactory<Program>` + `TestServer` for browser-based (Playwright)
tests. It seems appealing (official ASP.NET Core integration-test pattern) but it fundamentally
does not work for real browser navigation — see "Problems encountered" below. Instead:

1. Run the **real app** on a free local port, in a dedicated `Testing` environment — via Aspire's
   `DistributedApplicationTestingBuilder` if the user picked an AppHost, otherwise as a plain
   child process (`dotnet <dll>`).
2. Add a **test-only auth bypass** so Playwright doesn't need real credentials.
3. Add a **separate test database** — Aspire provisions its own container automatically; without
   Aspire, use an own connection string in `appsettings.Testing.json`.
4. Seed/reset data directly via the app's `DbContext` before each test.
5. Point Playwright at the running app's URL and drive the browser normally.

## Step 1 — Test-only auth bypass (skip if the user chose real login)

Add a fake `AuthenticationHandler` to the **app** project (not the test project, since `Program.cs`
must register it), guarded by environment so it's never active in Development/Staging/Production:

```csharp
// <AppProject>/Testing/TestAuthHandler.cs
using System.Security.Claims;
using System.Text.Encodings.Web;
using Microsoft.AspNetCore.Authentication;
using Microsoft.Extensions.Options;

namespace <AppNamespace>.Testing;

public class TestAuthHandler(
    IOptionsMonitor<AuthenticationSchemeOptions> options,
    ILoggerFactory logger,
    UrlEncoder encoder)
    : AuthenticationHandler<AuthenticationSchemeOptions>(options, logger, encoder)
{
    public const string SchemeName = "Test";

    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        var claims = new[] { new Claim(ClaimTypes.Name, "Playwright Test User") };
        var identity = new ClaimsIdentity(claims, SchemeName);
        var ticket = new AuthenticationTicket(new ClaimsPrincipal(identity), SchemeName);
        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}
```

In `Program.cs`, branch the real authentication registration on the environment:

```csharp
if (builder.Environment.IsEnvironment("Testing"))
{
    builder.Services.AddAuthentication(TestAuthHandler.SchemeName)
        .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>(TestAuthHandler.SchemeName, options => { });
}
else
{
    builder.Services.AddAuthentication(OpenIdConnectDefaults.AuthenticationScheme)
        .AddMicrosoftIdentityWebApp(builder.Configuration.GetSection("EntraId")); // or whatever the real scheme is
}
```

Also make sure DB migration and any Development-only middleware also runs (or is skipped, as
appropriate) for `Testing`:

```csharp
if (app.Environment.IsDevelopment() || app.Environment.IsEnvironment("Testing"))
{
    // migrate DB
}

if (!app.Environment.IsDevelopment() && !app.Environment.IsEnvironment("Testing"))
{
    app.UseExceptionHandler(...);
    app.UseHsts();
}
```

## Step 2 — CRITICAL: enable static web assets for the Testing environment

**This is the single most important gotcha in this whole skill.** Outside the `Development`
environment (or a truly published app), ASP.NET Core disables static web assets (CSS, and
crucially the Blazor/SignalR client scripts) unless explicitly re-enabled. If you skip this step,
the app renders its initial HTML fine (so a "page loads and shows text" test may even pass) but
**all client-side interactivity silently does nothing** — no button clicks, no dialogs, nothing —
because the Blazor Server script fails to load with a console error like:

```
Failed to load module script: Expected a JavaScript-or-Wasm module script but the server
responded with a MIME type of "". Strict MIME type checking is enforced...
```

...and the page shows an "An unhandled error has occurred. Reload" banner once you interact with
it. This is very confusing to debug from Playwright timeouts alone (looks like a locator/selector
problem). Fix it once, in `Program.cs`, before `builder.Build()`:

```csharp
if (builder.Environment.IsEnvironment("Testing"))
{
    builder.WebHost.UseStaticWebAssets();
}
```

## Step 3 — Separate test database config (skip if using the Aspire variant)

Add `appsettings.Testing.json` to the app project with its own connection string (different
database name from Development), so tests never touch real dev data:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=<AppName>Test;Trusted_Connection=true;TrustServerCertificate=true;"
  },
  "Logging": { "LogLevel": { "Default": "Warning", "Microsoft.AspNetCore": "Warning" } }
}
```

(With the Aspire variant, Aspire injects the real `ConnectionStrings:<dbResourceName>` for you —
`appsettings.Testing.json` only needs a `Logging` section, no connection string.)

The Web SDK copies `appsettings.*.json` to the output directory automatically — no csproj changes
needed.

## Step 4 — Create the test project

```powershell
dotnet new xunit -n <AppName>.PlaywrightTests -f net<TFM>
dotnet sln add <AppName>.PlaywrightTests
dotnet add <AppName>.PlaywrightTests package Microsoft.Playwright.Xunit
dotnet add <AppName>.PlaywrightTests reference <EfDataProject>.csproj   # if data seeding needs the DbContext directly
```

(For NUnit instead: `dotnet new nunit`, and `Microsoft.Playwright.NUnit` instead of
`Microsoft.Playwright.Xunit`.)

Then, depending on which Step 5 variant applies:

- **Aspire variant:** `dotnet add <AppName>.PlaywrightTests package Aspire.Hosting.Testing` and
  `dotnet add <AppName>.PlaywrightTests reference <AppProject>.AppHost.csproj` (**not** a reference
  to the app project itself — the AppHost project reference is what exposes the generated
  `Projects.<AppProject>_AppHost` type used below).
- **Process-launch variant:** `dotnet add <AppName>.PlaywrightTests reference <AppProject>.csproj`.

Do **not** add `Microsoft.AspNetCore.Mvc.Testing` in either variant — it's not needed (see
"Problems encountered" for why `WebApplicationFactory` fails here).

Also add a `playwright.runsettings` file in the test project folder so headed/slow-mo mode is
configurable without editing test code, and reference it from the csproj so both `dotnet test`
and Visual Studio's Test Explorer pick it up automatically (no manual "Configure Run Settings"
step, and no dependency on solution-root auto-detection):

```xml
<!-- <AppName>.PlaywrightTests/playwright.runsettings -->
<?xml version="1.0" encoding="utf-8"?>
<RunSettings>
  <Playwright>
    <BrowserName>chromium</BrowserName>
    <LaunchOptions>
      <!-- Defaults: headless, no delay. Override per-run via CLI without editing this file:
           dotnet test <path-to-csproj> -- Playwright.LaunchOptions.Headless=false Playwright.LaunchOptions.SlowMo=500 -->
      <Headless>true</Headless>
      <SlowMo>0</SlowMo>
    </LaunchOptions>
  </Playwright>
</RunSettings>
```

(Note: XML comments cannot contain `--`, so keep any comment describing the CLI override
argument-only, without the literal double-dash prefix.)

Reference it from the test project's csproj so Visual Studio Test Explorer and `dotnet test` both
use it automatically:

```xml
<PropertyGroup>
  ...
  <RunSettingsFilePath>$(MSBuildProjectDirectory)\playwright.runsettings</RunSettingsFilePath>
</PropertyGroup>
```

## Step 5 (Aspire variant) — Start the app via the existing AppHost

Prefer this variant whenever the solution already has an Aspire AppHost project. It starts real
container-backed dependencies (e.g. SQL Server) automatically instead of requiring LocalDB.

xUnit has no `[OneTimeSetUp]`/`[OneTimeTearDown]` equivalent — the idiomatic replacement is a
fixture class implementing `IAsyncLifetime`, shared across test classes via
`[CollectionDefinition]` + `ICollectionFixture<T>` so the app only starts once:

```csharp
// AppFixture.cs
using Aspire.Hosting;
using Aspire.Hosting.ApplicationModel;
using Aspire.Hosting.Testing;
using Xunit;

public class AppFixture : IAsyncLifetime
{
    private const string AppResourceName = "<resourceName>";      // name passed to builder.AddProject<...>("...") in AppHost.cs
    private const string DatabaseResourceName = "<dbResourceName>"; // name passed to .AddDatabase("...") in AppHost.cs
    private static readonly TimeSpan StartupTimeout = TimeSpan.FromMinutes(3);

    private DistributedApplication _app = null!;
    private HttpClient _httpClient = null!;
    public string BaseUrl => _httpClient.BaseAddress!.ToString().TrimEnd('/');

    public async Task InitializeAsync()
    {
        using var cts = new CancellationTokenSource(StartupTimeout);

        var appHost = await DistributedApplicationTestingBuilder
            .CreateAsync<Projects.<AppProject>_AppHost>(cts.Token);

        // Set the Testing environment directly on the resource — do NOT rely on passing
        // "--environment Testing" to CreateAsync and branching on it inside AppHost.cs; that
        // does not reliably propagate down to the project resource's own process (see
        // "Problems encountered").
        appHost.CreateResourceBuilder<ProjectResource>(AppResourceName)
            .WithEnvironment("ASPNETCORE_ENVIRONMENT", "Testing");

        _app = await appHost.BuildAsync(cts.Token);
        await _app.StartAsync(cts.Token);

        _httpClient = _app.CreateHttpClient(AppResourceName);
        await _app.ResourceNotifications.WaitForResourceHealthyAsync(AppResourceName, cts.Token);

        // Migrate explicitly rather than relying on the app's own startup migration — don't
        // assume ASPNETCORE_ENVIRONMENT reached the app process before this point (see
        // "Problems encountered").
        var connectionString = await _app.GetConnectionStringAsync(DatabaseResourceName, cts.Token);
        var options = new DbContextOptionsBuilder<AppDbContext>().UseSqlServer(connectionString).Options;
        await using var context = new AppDbContext(options);
        await context.Database.MigrateAsync(cts.Token);
    }

    public async Task DisposeAsync()
    {
        _httpClient?.Dispose();
        if (_app is not null) await _app.DisposeAsync();
    }

    public async Task ResetDataAsync(/* seed entities */)
    {
        var connectionString = await _app.GetConnectionStringAsync(DatabaseResourceName);
        var options = new DbContextOptionsBuilder<AppDbContext>().UseSqlServer(connectionString).Options;
        await using var context = new AppDbContext(options);
        // remove + re-add seed rows, SaveChangesAsync
    }
}

[CollectionDefinition(nameof(AppCollection))]
public class AppCollection : ICollectionFixture<AppFixture>;
```

```csharp
// PageTestBase.cs — thin base that delegates to the shared AppFixture
using Microsoft.Playwright.Xunit;

public abstract class PageTestBase(AppFixture fixture) : PageTest
{
    protected string BaseUrl => fixture.BaseUrl;
    protected Task ResetDataAsync(/* seed entities */) => fixture.ResetDataAsync(/* ... */);
}
```

(For NUnit instead: skip the fixture/collection classes above and use `[OneTimeSetUp]`/
`[OneTimeTearDown]` directly on a `PageTestBase : PageTest` from `Microsoft.Playwright.NUnit`, with
the same body as `AppFixture`'s `InitializeAsync`/`DisposeAsync`.)

Note this variant requires Docker to be running (for the SQL Server, or other, container) and the
first run can be noticeably slower than LocalDB while the container image is pulled/started.

## Step 5 (process-launch variant) — Launch the real app as a child process

Use this variant only when there is no Aspire AppHost in the solution.

```csharp
// TestAppServer.cs
using System.Net;
using System.Net.Sockets;
using System.Diagnostics;

public sealed class TestAppServer : IAsyncDisposable
{
    private Process? _process;
    public string BaseUrl { get; private set; } = string.Empty;

    public async Task StartAsync()
    {
        BaseUrl = $"http://127.0.0.1:{GetFreeTcpPort()}";

        var appDllPath = GetAppDllPath(); // see note below
        var startInfo = new ProcessStartInfo
        {
            FileName = "dotnet",
            Arguments = $"\"{appDllPath}\"",
            WorkingDirectory = Path.GetDirectoryName(appDllPath),
            RedirectStandardOutput = true,
            RedirectStandardError = true,
            UseShellExecute = false,
        };
        startInfo.EnvironmentVariables["ASPNETCORE_ENVIRONMENT"] = "Testing";
        startInfo.EnvironmentVariables["ASPNETCORE_URLS"] = BaseUrl;

        _process = Process.Start(startInfo) ?? throw new InvalidOperationException("Failed to start app.");

        using var httpClient = new HttpClient();
        var deadline = DateTime.UtcNow.AddSeconds(60);
        while (DateTime.UtcNow < deadline)
        {
            if (_process.HasExited)
            {
                var stderr = await _process.StandardError.ReadToEndAsync();
                throw new InvalidOperationException($"App exited early ({_process.ExitCode}): {stderr}");
            }
            try
            {
                var response = await httpClient.GetAsync(BaseUrl);
                if ((int)response.StatusCode < 500) return;
            }
            catch (HttpRequestException) { /* keep polling */ }
            await Task.Delay(250);
        }
        throw new TimeoutException("Timed out waiting for the app to start.");
    }

    private static int GetFreeTcpPort()
    {
        var listener = new TcpListener(IPAddress.Loopback, 0);
        listener.Start();
        var port = ((IPEndPoint)listener.LocalEndpoint).Port;
        listener.Stop();
        return port;
    }

    public ValueTask DisposeAsync()
    {
        if (_process is { HasExited: false }) _process.Kill(entireProcessTree: true);
        _process?.Dispose();
        return ValueTask.CompletedTask;
    }
}
```

**Important:** `WorkingDirectory` must be the *app's own* build output directory (its own
`bin/<Config>/<TFM>/`), **not** the test project's output directory, even though a `ProjectReference`
copies the app's `.dll`/`.deps.json`/`appsettings*.json` next to the test assembly. Running the
app's copied DLL from the test project's output directory still works functionally, but running it
from its own original build directory is simpler and avoids any doubt about static web assets /
content root resolution. Compute the app's own bin path relative to the test project's own
`AppContext.BaseDirectory` (same `<Config>/<TFM>` segments, sibling project folder), e.g.:

```csharp
var testBinDir = new DirectoryInfo(AppContext.BaseDirectory); // .../<TestProject>/bin/Debug/net10.0
var repoRoot = testBinDir.Parent!.Parent!.Parent!.Parent!.FullName;
var configuration = testBinDir.Parent!.Name; // Debug
var tfm = testBinDir.Name;                   // net10.0
var appDllPath = Path.Combine(repoRoot, "<AppProject>", "bin", configuration, tfm, "<AppProject>.dll");
```

## Step 6 (process-launch variant) — Base test fixture with DB seeding

Same xUnit `IAsyncLifetime` + `ICollectionFixture` pattern as the Aspire variant (see Step 5):

```csharp
// AppFixture.cs
using Xunit;

public class AppFixture : IAsyncLifetime
{
    private const string ConnectionString = "..."; // must match appsettings.Testing.json
    private TestAppServer _server = null!;
    public string BaseUrl => _server.BaseUrl;

    public async Task InitializeAsync()
    {
        _server = new TestAppServer();
        await _server.StartAsync();
    }

    public async Task DisposeAsync() => await _server.DisposeAsync();

    public async Task ResetDataAsync(/* seed entities */)
    {
        var options = new DbContextOptionsBuilder<AppDbContext>().UseSqlServer(ConnectionString).Options;
        await using var context = new AppDbContext(options);
        // remove + re-add seed rows, SaveChangesAsync
    }
}

[CollectionDefinition(nameof(AppCollection))]
public class AppCollection : ICollectionFixture<AppFixture>;
```

```csharp
// PageTestBase.cs
using Microsoft.Playwright.Xunit;

public abstract class PageTestBase(AppFixture fixture) : PageTest
{
    protected string BaseUrl => fixture.BaseUrl;
    protected Task ResetDataAsync(/* seed entities */) => fixture.ResetDataAsync(/* ... */);
}
```

(For NUnit instead: fold `AppFixture`'s body directly into a `PageTestBase : PageTest` using
`[OneTimeSetUp]`/`[OneTimeTearDown]` — no separate fixture/collection classes needed.)

Run `playwright install chromium` (or `pwsh bin/<Config>/<TFM>/playwright.ps1 install chromium`)
after the first build to download the browser binaries. This is needed regardless of which Step 5
variant is used.

## Step 7 — Write the actual page tests

Add `[Collection(nameof(AppCollection))]` on each concrete test class, take `AppFixture` via its
constructor and pass it to `: PageTestBase(fixture)`. Per-test setup (the NUnit `[SetUp]`
equivalent) goes in an `override` of `PageTest`'s own `InitializeAsync()` — it's declared
`public virtual`/`override` in `Microsoft.Playwright.Xunit.PageTest` specifically so subclasses can
extend it — calling `base.InitializeAsync()` first:

```csharp
[Collection(nameof(AppCollection))]
public class PersonListTests(AppFixture fixture) : PageTestBase(fixture)
{
    public override async Task InitializeAsync()
    {
        await base.InitializeAsync(); // creates Page/BrowserContext — must run first
        await ResetDataAsync(/* seed rows for this test class */);
    }

    [Fact]
    public async Task ExampleTest()
    {
        await Page.GotoAsync($"{BaseUrl}/persons");
        // ...
    }
}
```

Prefer `Page.GetByRole(...)` and `Page.GetByLabel(...)` locators. See "Problems encountered" for
required workarounds around dialogs, duplicate labels, and Blazor Server's binding timing.

---

## Problems encountered (read this before debugging test failures)

These are real failures hit while building this kind of test project, roughly in the order
they'd bite:

1. **`WebApplicationFactory<Program>` + Kestrel does not work for Playwright.** The official
   ASP.NET Core testing pattern uses `TestServer`, which has no real TCP socket for a browser to
   connect to. Trying to make `CreateHost` build a second, real Kestrel-backed host (a commonly
   suggested workaround: build once for `TestServer`, once for Kestrel) **re-runs the app's
   `Program.cs` top-level statements twice** (including EF migrations), causing concurrent-DDL SQL
   errors ("A severe error occurred on the current command"). Returning only a Kestrel host from
   `CreateHost` instead throws `InvalidCastException: Unable to cast ... KestrelServerImpl ...
   to ... TestServer` the moment anything touches `factory.Services` or `factory.Server`, because
   `WebApplicationFactory` unconditionally requires a `TestServer`-backed host for those
   accessors. **Conclusion: don't use `WebApplicationFactory` at all for this** — use the Aspire
   variant if available, otherwise launch the app as a real out-of-process `dotnet <dll>` process
   (see Step 5).

2. **Aspire: passing `--environment Testing` as an arg to `DistributedApplicationTestingBuilder.
   CreateAsync` and branching on `builder.Environment.IsEnvironment("Testing")` inside `AppHost.cs`
   to conditionally set the project resource's `ASPNETCORE_ENVIRONMENT` does not reliably work** —
   the app resource ended up running in its default environment, so environment-gated code (auth
   bypass, static web assets, migration) in the app's own `Program.cs` never activated, and
   `Person`/other tables didn't exist yet (`Invalid object name 'X'` SQL errors during seeding).
   Fix: set the environment variable directly on the resource from the *test*, via
   `appHost.CreateResourceBuilder<ProjectResource>(resourceName).WithEnvironment("ASPNETCORE_ENVIRONMENT", "Testing")`
   before calling `BuildAsync`, and don't rely on the AppHost's own environment at all.

3. **Aspire: don't assume the app's own conditional migration-on-startup will have finished by the
   time `WaitForResourceHealthyAsync` returns**, even with the environment variable fix above —
   safer to run `context.Database.MigrateAsync()` explicitly in the test's `OneTimeSetUp` once the
   connection string is available via `app.GetConnectionStringAsync(dbResourceName)`, rather than
   depend on the app's internal `Program.cs` migration logic firing correctly in time.

4. **xUnit has no `[OneTimeSetUp]`/`[OneTimeTearDown]`/`[SetUp]` equivalents — don't try to bolt
   them onto a plain base class.** The idiomatic xUnit replacement for "start this expensive
   resource once and share it across many tests" is a fixture class implementing `IAsyncLifetime`
   registered via `[CollectionDefinition(nameof(X))]` + `ICollectionFixture<TFixture>`, with test
   classes opting in via `[Collection(nameof(X))]` and receiving the fixture through their
   constructor. Trying to reuse the NUnit-style single `PageTestBase` (holding both the one-time
   app startup *and* the per-test reset logic) doesn't map cleanly — split them: a separate
   `AppFixture : IAsyncLifetime` for the once-per-collection app startup, and the test class's own
   `override InitializeAsync()` for per-test reset/seeding (see Steps 5–7).

5. **`Microsoft.Playwright.Xunit.PageTest.InitializeAsync()` is `public virtual`/`override`, so it
   can safely be overridden** in a test class to add per-test setup (e.g. DB seeding) — just call
   `base.InitializeAsync()` **first** so `Page`/`Context` exist before your own code runs. This
   isn't obvious from the docs; confirmed by reading the actual
   [Playwright.Xunit/PageTest.cs source](https://github.com/microsoft/playwright-dotnet/blob/main/src/Playwright.Xunit/PageTest.cs)
   before relying on it.

6. **Static web assets are silently disabled outside Development / non-published apps.** Symptom:
   the page's initial HTML renders fine (so a simple "page shows text" test can even pass), but
   every button click / dialog / interactive feature does nothing. Browser console shows
   `Failed to load module script... MIME type of ""`, and the page displays "An unhandled error
   has occurred." This is because ASP.NET Core only auto-wires static web assets in `Development`
   or when running truly published output; a `Testing` environment gets neither. Fix:
   `builder.WebHost.UseStaticWebAssets();` guarded to the `Testing` environment (Step 2). Don't
   waste time inspecting Playwright locators/selectors before ruling this out — check the browser
   console for module-script/MIME errors first.

7. **Running the app's DLL from the wrong working directory breaks config.** If you invoke
   `dotnet <path>\App.dll` with a `WorkingDirectory`/cwd that isn't the app's own bin output
   folder, `appsettings.{Environment}.json` won't be found (ContentRoot defaults to the process's
   current directory, not the DLL's directory), leading to "No connection string configured" or
   similar startup exceptions. Always set `ProcessStartInfo.WorkingDirectory` explicitly to the
   directory containing the app's DLL.

8. **Duplicate ARIA labels across a page and an open dialog cause ambiguous locators.** E.g. a
   search form's "Name" field and an "Add new person" dialog's "Name" field both exist in the DOM
   at once. Scope dialog interactions with `Page.GetByRole(AriaRole.Dialog)` (or a dialog-specific
   locator) rather than querying the whole page, e.g. `dialog.GetByLabel("Name")` instead of
   `Page.GetByLabel("Name")`. Don't rely on a guessed CSS class like `.mud-dialog` — it can be
   wrong for a given component-library version; the ARIA `dialog` role is more stable and was
   confirmed to render correctly here.

9. **Blazor Server field bindings without `Immediate="true"` only commit on blur, not on
   Playwright's `FillAsync`.** `FillAsync` sets the value and dispatches an `input` event but does
   not blur the field. If a bound field (e.g. a MudBlazor `MudTextField` without
   `Immediate="true"`) only listens for `onchange` (blur), a subsequent button click (e.g. "Save")
   can read a stale/empty value, leaving a required-field-gated button permanently disabled, or a
   search filter silently not applied. Fix in the test: press `Tab` (or otherwise blur) after
   filling such fields and before asserting on dependent state:
   ```csharp
   await field.FillAsync("value");
   await Page.Keyboard.PressAsync("Tab");
   ```
   This affects **any** Blazor Server app using non-immediate two-way bound inputs, not just
   MudBlazor.

10. **A freshly clicked button that opens a dialog can occasionally not open on the very first
   click** in a fresh page/circuit (observed intermittently when running the full suite vs. a
   single test in isolation). Guard with a short wait-then-retry rather than a bare
   `ClickAsync()` immediately followed by an assumption the dialog is open:
   ```csharp
   await addButton.ClickAsync();
   try { await dialogField.WaitForAsync(new LocatorWaitForOptions { Timeout = 5000 }); }
   catch (TimeoutException) { await addButton.ClickAsync(); } // retry once
   ```

11. **`AppDbContext` `DbSet` property names may not match the entity name.** E.g. a `Person`
   entity's `DbSet` might be exposed as `People`, not `Persons`. Always check the actual
   `DbContext` before writing seeding code — don't assume pluralization.

12. **`Microsoft.AspNetCore.Authentication` (for `AuthenticationSchemeOptions`) must be imported in
   `Program.cs`** when adding the conditional test auth scheme — easy to miss and causes a
   confusing `CS0246` if forgotten.
