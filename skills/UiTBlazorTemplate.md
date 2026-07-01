---
name: dotnetdocs-bootstrap
description: "Bootstrap and standardize a .NET 10 Blazor + Aspire solution based on DotNetDocs chat history. Assumes solution file already exists. Prompts for app name, Entra TenantId, and Entra ClientId before implementation, and enforces established UI/auth design across key files."
prerequsites: Empty solution created. 
license: MIT
metadata:
  author: GitHub Copilot
  version: "1.1.0"
  sourceChats:
    - chatlogs/chat-2026-07-01.md
    - chatlogs/ChatLog5.md
    - chatlogs/ChatLog24.md
---

# DotNetDocs Bootstrap Skill

## Purpose
Apply the established implementation and design pattern from previous DotNetDocs chats to a fresh repo state.

## Assumption
- The user has already created the solution file (`.slnx`/`.sln`).

## Required questions (must ask first)
1. What should the app name be?
2. What is the Microsoft Entra ID `TenantId`?
3. What is the Microsoft Entra ID `ClientId`?

Do not proceed until these are answered.

## Expected solution shape
- `src/<AppName>.AppHost` (Aspire AppHost)
- `src/<AppName>.ServiceDefaults` (service defaults / telemetry)
- `src/<AppName>` (Blazor .NET 10 app)
- Optional: `tests/<AppName>.Tests` with TUnit

## Implementation workflow
1. Validate solution/projects exist and are included in the solution file.
2. Ensure AppHost registers the web project via `builder.AddProject<Projects.<AppName>>("<appname-lower>");`.
3. Add required NuGet packages for the web app:
   - `MudBlazor`
   - `Microsoft.Identity.Web`
4. Implement UI/auth design conventions in the exact files listed below.
5. Configure authentication/authorization in `Program.cs`:
   - `AddAuthentication().AddMicrosoftIdentityWebApp(...)`
   - `AddAuthorization` with fallback policy
   - `UseAuthentication`, `UseAuthorization`
   - `MapGroup("/authentication").MapLoginAndLogout()`
   - Require auth for app routes (`MapRazorComponents<App>().RequireAuthorization()`)
6. Configure app settings:
   - Set `AzureAd:Instance`, `AzureAd:TenantId`, `AzureAd:ClientId`, `CallbackPath`
   - Keep production placeholders in `appsettings.json`
   - Set real values in `appsettings.Development.json`
7. Add tests project with TUnit when requested:
   - Target `net10.0`
   - Enable Microsoft Testing Platform support via `global.json` test runner
   - Add at least service-level tests for UI state services

## Design specifications from prior chats (must preserve)

### `Services/AppUiService.cs`
- Class name: `AppUiService`.
- Lifetime: singleton.
- Must expose:
  - `AppTitle` defaulting to `"DotNet Docs"`.
  - `MudTheme MyTheme` with both `PaletteLight` and `PaletteDark` configured.
- Theme values can use placeholder UiT colors, but include both app bar and drawer colors.
- `AppTitle` and theme belong here, not in `PersonalUiService`.

### `Services/PersonalUiService.cs`
- Class name: `PersonalUiService`.
- Lifetime: scoped (per user/circuit).
- Must own user-specific UI state:
  - `DrawerOpen` (default `true`) with change notification.
  - `IsDarkTheme` with change notification.
  - `UserPrincipalName` for authenticated display.
- Must expose `DrawerToggle()` and `event Action? OnChange`.
- Property setters should call a central notify method.

### `Components/App.razor`
- Include MudBlazor CSS and JS assets.
- Keep Google Roboto font link.
- Keep UiT external favicon links block.
- Use interactive server render mode:
  - `<HeadOutlet @rendermode="@InteractiveServer" />`
  - `<Routes @rendermode="@InteractiveServer" />`

### `Components/Layout/MainLayout.razor`
- Use MudBlazor layout primitives (`MudLayout`, `MudAppBar`, `MudDrawer`, `MudMainContent`).
- AppBar should include:
  - menu button bound to drawer toggle
  - logo image `Src="/images/uit_en.png"`
  - app title from `AppUiService.AppTitle`
  - `LoginDisplay` and `ToggleDarkLightMode`
- Drawer behavior:
  - responsive/clipped MudDrawer
  - bind open state to `personalUi.DrawerOpen`
- Must include providers at bottom:
  - `<MudPopoverProvider />`
  - `<MudSnackbarProvider />`
  - `<MudDialogProvider />`
- Components should subscribe/unsubscribe to `personalUi.OnChange` safely.

### `Components/Layout/NavMenu.razor`
- MudBlazor nav only (no bootstrap nav markup).
- Wrap menu in `<MudPaper Class="py-3 mt-0" Elevation="0">`.
- Include base links (Home/Counter/Weather), divider, and grouped fake links:
  - `MudDivider DividerType="DividerType.Middle"`
  - `MudNavGroup Title="Fake links" Icon="@Icons.Material.Filled.LocationCity" HideExpandIcon="false"`

### `Components/Layout/LoginDisplay.razor` (+ `.razor.css`)
- Use `AuthorizeView` with login/logout behavior:
  - unauthenticated -> navigate to `authentication/login`
  - authenticated -> POST form to `authentication/logout` + antiforgery token + return URL
- Populate `personalUi.UserPrincipalName` from `ClaimTypes.Upn` fallback to identity name.
- Keep styled wrappers (`auth_div`, `login_div`, `logout_div`) and button/form classes from prior design.

### `Components/Layout/ToggleDarkLightMode.razor`
- Host `MudThemeProvider` bound to `personalUi.IsDarkTheme`.
- Toggle icon switches dark/light state through `PersonalUiService`.
- Subscribe/unsubscribe to `personalUi.OnChange` to keep UI synchronized.

### `Program.cs`
- Register service lifetimes exactly:
  - `AddScoped<PersonalUiService>()`
  - `AddSingleton<AppUiService>()`
- Add `AddMudServices()` and `AddCascadingAuthenticationState()`.
- Enforce authenticated site via fallback policy and `.RequireAuthorization()` on mapped components.

### `DocsApp.csproj` (or app project `.csproj`)
- Ensure image content copy rule exists:
  - `wwwroot\images\**` with `CopyToOutputDirectory=PreserveNewest`.

## Mandatory upgrade sequence
After adding projects and NuGet packages:
1. Update all NuGet packages to latest compatible versions.
2. Upgrade Aspire CLI to latest stable.
3. Upgrade Aspire AppHost SDK/packages to latest stable.
4. Rebuild and verify.

## Validation checklist
- Solution build succeeds.
- Authentication challenge redirects to Entra sign-in.
- Logout endpoint works.
- MudBlazor layout renders and drawer/theme toggles function.
- AppHost project appears in Aspire dashboard.
- Tests pass (if test project exists).
- `MainLayout`, `AppUiService`, `PersonalUiService`, and `App.razor` match this skill's design contract.

## Notes from prior chats
- Keep logo path as `/images/uit_en.png` and ensure image copy rules are set.
- Use UiT favicon links in `App.razor` (external URLs).
- For .NET 10 + TUnit, configure Microsoft Testing Platform runner in `global.json`.
