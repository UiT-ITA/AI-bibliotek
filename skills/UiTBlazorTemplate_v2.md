---
name: UiT BlazorTemplate
description: "Bootstrap and standardize a .NET 10 Blazor + Aspire solution based on UiTBlazorTemplate chat history. Assumes solution file already exists. Prompts for app name, Entra TenantId, and Entra ClientId before implementation, and enforces established UI/auth design across key files."
prerequsites: Empty solution created. 
license: MIT
metadata:
  author: GitHub Copilot
  version: "1.2.0"
  sourceChats:
    - chatlogs/chat-2026-07-01.md
    - chatlogs/ChatLog5.md
    - chatlogs/ChatLog24.md
---

- DO NOT DEBUG AND CORRECT ERRORS, I AM DEBUGGING THIS SKILL AND WANT CORRECT MISTAKES

- Enable nullable reference types and treat warnings seriously.
- Add  .editorconfig  for consistent formatting and naming.
- Use analyzers ( StyleCop ,  Microsoft.CodeAnalysis.NetAnalyzers ) and consider warnings as errors.
- Keep project structure simple: separate  src/  and  tests/ , one project per concern.
- Use xUnit for tests, and run them from Test Explorer.
- Prefer async/await for I/O-bound work.
- Use dependency injection and avoid hard-coded dependencies.
- Manage packages with NuGet and pin versions where possible.
- Use Git integration in Visual Studio, with small commits and pull requests.


# UiT MudBlazor guidance
## Required inputs (ask first)
Do not proceed past planning until all required inputs are provided.
Before making any code/config changes, collect these values from the user:

1. `applicationName`
2. `tenantId`
3. `clientId`
4. `solutionFileName`, default is applicationName with .slx file extension
5. `applicationVersion`, default '1.0.0'

Rules:
- Do not assume or invent defaults.
- If one or more values are missing, ask only for the missing values.
- Wait for the user’s response before continuing.
- Reuse provided values consistently across generated code, config, and documentation.

## Overall guidance
- Core C# rules:  .NET 10+ , nullable enabled, analyzers on,  EditorConfig  enforced.
- Project conventions: folder layout, naming, error handling, async/await, DI, testing style.
- Keep project structure simple: separate  src/ and tests/ , one project per concern.
- Framework-specific skills if relevant:
- ASP.NET Core API: controllers/minimal APIs, validation, auth, OpenAPI.
- EF Core: migrations, DbContext patterns, query conventions.
- Azure: App Service, Functions, Azure Monitor App Insights, deployment.
- Testing: xUnit/NUnit, mocking, integration tests.

## Stack
- Use .NET 10+ and SDK-style .csproj  files
- Use OpenTelemetry for tracing, metrics, and logs where supported.
- Use Microsoft Entra ID for authentication and authorization.

## Scope
- Use MudBlazor for all Blazor UI unless there is a strong reason not to.
- Prefer consistent, reusable UI patterns over one-off markup.
- Keep components simple and declarative.

## Components
- Prefer MudLayout, MudAppBar, MudDrawer, MudMainContent for app shell.
- Prefer MudGrid, MudStack, MudCard, MudPaper for layout and grouping.
- Prefer MudTextField, MudSelect, MudAutocomplete, MudSwitch, MudCheckbox for inputs.
- Prefer MudButton, MudIconButton, MudMenu, MudTabs, MudTable, MudDialog, MudSnackbar for interaction.
- Prefer MudProgressCircular, MudSkeleton, MudAlert for loading and feedback.

## Forms
- Use EditForm with validation.
- Surface validation errors inline.
- Keep labels, placeholders, and helper text clear and consistent.
- Prefer built-in MudBlazor validation patterns before custom logic.

## Styling
- Prefer MudBlazor parameters, theme, spacing, and typography before custom CSS.
- Use custom CSS only when MudBlazor parameters are insufficient.
- Avoid inline styles unless necessary.

## Data and state
- Load data asynchronously.
- Show loading, empty, and error states explicitly.
- Keep state logic out of the UI when it can live in a service or view model.

## Logging
- Avoid the use of `Console.WriteLine`.
- Inject `ILogger<T>` into every service/component that needs logging.
- Log structured events with placeholders, not string interpolation.
- Use appropriate levels:
  - `LogInformation` for normal milestones
  - `LogWarning` for recoverable issues
  - `LogError` for failures
- Never swallow exceptions silently.

## Authentication and authorization
- Use Entra ID authentication via ASP.NET Core authentication middleware.
- Protect pages/endpoints with `[Authorize]`.
- Use policies and roles for authorization, not ad hoc checks.
- Prefer claims-based checks when role checks are too coarse.

## OpenTelemetry
- Configure OpenTelemetry in `Program.cs`.
- Use a single shared service identity:
  - `serviceName`: `ApplicationName`
  - `serviceVersion`: `1.0.0`
  - `serviceInstanceId`: `Environment.MachineName`
- Instrument ASP.NET Core, HttpClient, and app-specific dependencies consistently.

## Program.cs example
```csharp
#region OpenTelemetry
builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource.AddService(
        serviceName: "ApplicationName",
        serviceVersion: "ApplicationVersion",
        serviceInstanceId: Environment.MachineName))
    .WithTracing(tracing =>
    {
        tracing.AddAspNetCoreInstrumentation();
        tracing.AddHttpClientInstrumentation();
    })
    .WithMetrics(metrics =>
    {
        metrics.AddAspNetCoreInstrumentation();
        metrics.AddHttpClientInstrumentation();
    });
#endregion
```

## Components guidance
- Scoped class for user information and UI, classname UIUser
- Singleton class for application UI, classname UIApp

## UIUser example
```csharp
public class UIUser
{
   public string? UserPrincipalName { get; set; } = null;
}
```

## UIApp example
```csharp
public class UIApp
{  
  public UIApp(IConfiguration config)
  {
    AppTitle = config?.GetSection("AppVariables")?.GetValue<string>("AppTitle") ?? "Application Title";
    appTheme = GetTheme();
  }

  // Get AppTitle from config
  public string AppTitle { get; private set; } = "Application Title";

  #region Theming
  public MudTheme appTheme { get; private set; } = null!;

  private MudTheme GetTheme = new()
  {
      PaletteLight = new PaletteLight()
      {
          //    Black                       = "#272c34ff",
          //    BackgroundGrey              = "#27272f",
          //    Surface                     = "#ffffff",
          //    Background                  = "#faf9f8",
          //    DrawerText                  = "rgba(44,43,42,255)",
          //    DrawerBackground            = "#f0f0f0",
          //    DrawerIcon                  = "rgba(0,0,0, 0.50)",
          LinesInputs = "rgba(200,200,200, 0.32)",            // Linjer i tekstbokser
          AppbarBackground = "#043348",                            // App bar
          AppbarText = "rgba(255,255,255, 0.70)",            // App bar text
          TextPrimary = "rgba(0,0,0, 0.90)",
          TextSecondary = "rgba(190,190,190, 0.50)",
          //    TextSecondary               = "rgba(60,60,60, 0.50)",         // f.eks. inne i tekstbokser
          //    ActionDefault               = "#adadb1",
          //    ActionDisabled              = "rgba(255,255,255, 0.26)",
          //    ActionDisabledBackground    = "rgba(255,255,255, 0.12)"
          Success = "#4caf50"
      },
      PaletteDark = new PaletteDark()
      {
          //    Black                       = "#272c34ff",
          //    BackgroundGrey              = "#27272f",
          //    Surface                     = "#ffffff",
          //    Background                  = "#faf9f8",
          //    DrawerText                  = "rgba(44,43,42,255)",
          DrawerBackground = "#373740",
          //    DrawerIcon                  = "rgba(0,0,0, 0.50)",
          LinesInputs = "rgba(20,20,20, 0.32)",               // Linjer i tekstbokser
          AppbarBackground = "#333333",                            // App bar
          AppbarText = "rgba(50,50,50, 0.70)",               // App bar text
          TextPrimary = "rgba(170,170,170, 0.90)",
          TextSecondary = "rgba(190,190,190, 0.50)",
          // App bar text
          //    ActionDefault               = "#adadb1",
          //    ActionDisabled              = "rgba(255,255,255, 0.26)",
          //    ActionDisabledBackground    = "rgba(255,255,255, 0.12)"
          Success = "#344234"
      },
      Shadows = new Shadow(),
      ZIndex = new ZIndex(),
      LayoutProperties = new LayoutProperties()
      {
          DrawerWidthLeft = "250px",        // Størrelse på meny
          DrawerWidthRight = "1100px",      // Størrelse på drawer til høyre
          AppbarHeight = "80px",            // Høyden på logolinjen på topp
          DrawerMiniWidthLeft = "15px"      // Størrelse på meny når den er minimert
      },
      Typography = new Typography()
      {
          Default = new DefaultTypography() { FontFamily = ["Open Sans", "Roboto", "Helvetica", "Arial", "sans-serif"], FontSize = "0.875rem", FontWeight = "400", LineHeight = "1.43", LetterSpacing = ".01071em", TextTransform = "none" },
          H1 = new H1Typography() { FontFamily = ["Open Sans", "Roboto", "Helvetica", "Arial", "sans-serif"], FontSize = "2.000rem", FontWeight = "300", LineHeight = "1.15", LetterSpacing = "-.01562em", TextTransform = "none" },
          H2 = new H2Typography() { FontFamily = ["Open Sans", "Roboto", "Helvetica", "Arial", "sans-serif"], FontSize = "1.700rem", FontWeight = "300", LineHeight = "1.15", LetterSpacing = "-.00833em", TextTransform = "none" },
          H3 = new H3Typography() { FontFamily = ["Open Sans", "Roboto", "Helvetica", "Arial", "sans-serif"], FontSize = "1.500rem", FontWeight = "300", LineHeight = "1.15", LetterSpacing = "0", TextTransform = "none" },
          H4 = new H4Typography() { FontFamily = ["Open Sans", "Roboto", "Helvetica", "Arial", "sans-serif"], FontSize = "1.300rem", FontWeight = "300", LineHeight = "1.15", LetterSpacing = ".0075em", TextTransform = "none" },
          H5 = new H5Typography() { FontFamily = ["Open Sans", "Roboto", "Helvetica", "Arial", "sans-serif"], FontSize = "1.100rem", FontWeight = "300", LineHeight = "1.15", LetterSpacing = "0", TextTransform = "none" },
          H6 = new H6Typography() { FontFamily = ["Open Sans", "Roboto", "Helvetica", "Arial", "sans-serif"], FontSize = "1.000rem", FontWeight = "300", LineHeight = "1.15", LetterSpacing = ".0075em", TextTransform = "none" },
          Subtitle1 = new Subtitle1Typography() { FontFamily = ["Open Sans", "Roboto", "Helvetica", "Arial", "sans-serif"], FontSize = "1.000rem", FontWeight = "400", LineHeight = "1.75", LetterSpacing = ".00938em", TextTransform = "none" },
          Subtitle2 = new Subtitle2Typography() { FontFamily = ["Open Sans", "Roboto", "Helvetica", "Arial", "sans-serif"], FontSize = "0.875rem", FontWeight = "500", LineHeight = "1.57", LetterSpacing = ".01em", TextTransform = "none" },
          Body1 = new Body1Typography() { FontFamily = ["Open Sans", "Roboto", "Helvetica", "Arial", "sans-serif"], FontSize = "0.900rem", FontWeight = "400", LineHeight = "1.50", LetterSpacing = ".00938em", TextTransform = "none" },
          Body2 = new Body2Typography() { FontFamily = ["Open Sans", "Roboto", "Helvetica", "Arial", "sans-serif"], FontSize = "0.750rem", FontWeight = "400", LineHeight = "1.75", LetterSpacing = ".01071em", TextTransform = "none" },
          Button = new ButtonTypography() { FontFamily = ["Open Sans", "Roboto", "Helvetica", "Arial", "sans-serif"], FontSize = "1.000rem", FontWeight = "500", LineHeight = "1.75", LetterSpacing = ".02857em", TextTransform = "none" },
          Caption = new CaptionTypography() { FontFamily = ["Open Sans", "Roboto", "Helvetica", "Arial", "sans-serif"], FontSize = "0.750rem", FontWeight = "400", LineHeight = "1.66", LetterSpacing = ".03333em", TextTransform = "none" },
          Overline = new OverlineTypography() { FontFamily = ["Open Sans", "Roboto", "Helvetica", "Arial", "sans-serif"], FontSize = "0.750rem", FontWeight = "400", LineHeight = "2.66", LetterSpacing = ".08333em", TextTransform = "none" }
      }
  };
  #endregion
}
```


## MainLayout.razor example
```
﻿@inherits LayoutComponentBase

<MudThemeProvider Theme="uiApp.MudTheme" />
<MudDialogProvider />
<MudSnackbarProvider />
<MudPopoverProvider />
<Spinner />
    <MudLayout>
        <MudAppBar Elevation="1" Dense="true">
            <div class="d-none d-md-block">
                <MudIconButton Icon="@Icons.Material.Filled.MenuOpen" Color="Color.Tertiary" Edge="Edge.Start" OnClick="@(() => DrawerToggle())" />
                <MudImage Alt="The artic University of Norway" Src="/images/uit_en.png" Class="ml-2 mr-8"></MudImage>
            </div>
            <MudText Class="ml-3" Style="font-size:1.5em;font-weight:500;color:#f5f5f5">@uiApp.AppTitle</MudText>
            <MudSpacer />
            <div class="d-none d-md-block">
                <LoginDisplay />
            </div>
            <ToggleDarkLightMode />       
        </MudAppBar>
        <MudDrawer @bind-Open="_drawerOpen" ClipMode="DrawerClipMode.Always" Variant="@DrawerVariant.Responsive" OpenMiniOnHover="false" Elevation="2">
          @if(_drawerOpen)
          { <NavMenu /> }
          else
          { <div style="width:100%;height:100%;"></div> }
        </MudDrawer>
        <MudMainContent>
          <div style="padding-block-end:180px;" class="pt-4 pl-8 pr-4">
              @Body
          </div>
        </MudMainContent>
    </MudLayout>

<div id="blazor-error-ui">
    An unhandled error has occurred.
    <a href="" class="reload">Reload</a>
    <a class="dismiss">🗙</a>
</div>

@code {
    [Inject] UIApp uiApp { get; set; } = null!;
    bool _drawerOpen = true;
    void DrawerToggle() => _drawerOpen = !_drawerOpen; 
}
```
## NavMenu.razor example
```
<MudPaper Width="250px mt-0" Class="py-3" Elevation="0">
    <MudNavMenu>
        <MudNavLink Match="NavLinkMatch.All" Icon="@Icons.Material.Filled.LibraryBooks" Href="/">Hjem</MudNavLink>
        <MudDivider DividerType="DividerType.Middle" Class="mt-3 mb-3"></MudDivider>
        <MudNavLink Href="/weather" Icon="@Icons.Material.Filled.Phone">Test1</MudNavLink>
        <MudNavLink Href="/button" Icon="@Icons.Material.Filled.CreditCard">Adgangskort</MudNavLink>

        <MudDivider DividerType="DividerType.Middle"></MudDivider>
        <MudNavLink Href="/kontoradresse" Icon="@Icons.Material.Filled.PersonSearch">Kontoradresse</MudNavLink>

        <MudDivider DividerType="DividerType.Middle" Class="mt-3 mb-3"></MudDivider>     
        <MudText Typo="Typo.button" Class="pa-4">Testing</MudText>
        <MudNavLink Match="NavLinkMatch.All" Href="/notexist">404 error</MudNavLink>
        <MudNavLink Match="NavLinkMatch.All" Href="/notauthorized_page">Ikke tilgang</MudNavLink>        
        <MudDivider DividerType="DividerType.Middle" Class="mt-3 mb-3"></MudDivider>
        <MudNavLink Match="NavLinkMatch.All" Href="/claims">Claims</MudNavLink>
        <MudNavLink Match="NavLinkMatch.All" Href="/typo">Typography</MudNavLink> 
        <MudNavLink Match="NavLinkMatch.All" Href="/about">Om Min Info</MudNavLink>
    </MudNavMenu>
</MudPaper>

@code {
}
```

## ToggleDarkLightMode.razor example
```
﻿<MudThemeProvider Theme="uiApp.MyTheme" @bind-IsDarkMode="@uiUser.IsDarkTheme" />
@if (myUI.IsDarkTheme)
{
    <MudIconButton Icon="@Icons.Material.Filled.LightMode" Style="@($"color:{myUI.MyTheme.PaletteLight.GrayLight};")" OnClick="@((e) => SwitchDarkLightMode())" />
}
else
{
    <MudIconButton Icon="@Icons.Material.Filled.DarkMode" Style="@($"color:{myUI.MyTheme.PaletteDark.GrayLight};")" OnClick="@((e) => SwitchDarkLightMode())" />
}

@code {
    [Inject] UIApp uiApp { get; set; } = null!;
    [Inject] UIUser uiUser { get; set; } = null!;

    void SwitchDarkLightMode()
    {
        myUI.SwitchTheme();
        this.StateHasChanged();
    }
}
```

## LoginDisplay.razor example
```
@implements IDisposable

<AuthorizeView>
    <Authorized>
        <div class="auth_div logout_div">
            <form action="authentication/logout" method="post">
                <AntiforgeryToken />
                <input type="hidden" name="ReturnUrl" value="@currentUrl" />
                <button type="submit" >
                    Logout @uiUser.UserPrincipalName
                </button>
            </form>
        </div>
    </Authorized>
    <NotAuthorized>
        <div class="auth_div login_div" @onclick="GotoLogin">Login</div>
    </NotAuthorized>
</AuthorizeView>


@code {
    [Inject] public required UIUser uiUser { get; set; } 
    [Inject] public required NavigationManager NavigationManager { get; set; }
    [Inject] public required AuthenticationStateProvider _auth { get; set; }
    private string? currentUrl;

    protected override async Task OnInitializedAsync()
    {
//        if (uitUser.UserPrincipalName == string.Empty)
//        {
//            var authState = await _auth.GetAuthenticationStateAsync();
//            var user = authState.User;
//            if (user.Identity is not null && user.Identity.IsAuthenticated)
//            {
//                if (user.Identity.Name is null || user.Identity.Name == string.Empty)
//                    throw new Exception("User.Identity.Name is null or empty");
//                //"http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn"
//                string? fasit = user.FindFirst("http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn")?.Value ?? user.Identity?.Name;
//                if (fasit is null) throw new Exception("Could not determine username");
//                uitUser.UserPrincipalName = fasit;
//                //uitUser.UserPrincipalName = user.Identity.Name;
//            }
//        }
        currentUrl = NavigationManager.ToBaseRelativePath(NavigationManager.Uri);
        NavigationManager.LocationChanged += OnLocationChanged;
        await base.OnInitializedAsync();
    }


    private void GotoLogin()
    {
        NavigationManager.NavigateTo("authentication/login", true);
    }

    private void OnLocationChanged(object? sender, LocationChangedEventArgs e)
    {
        currentUrl = NavigationManager.ToBaseRelativePath(e.Location);
        StateHasChanged();
    }

    public void Dispose()
    {
        NavigationManager.LocationChanged -= OnLocationChanged;
    }
}

```




## Code quality
• Prefer DI over static helpers.
• Prefer async APIs.
• Keep components small and focused.
• Reuse existing patterns in the repo before introducing new abstractions.



