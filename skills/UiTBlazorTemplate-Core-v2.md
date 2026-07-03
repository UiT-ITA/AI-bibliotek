---
name: UiTBlazorTemplate-Core
description: "Bootstrap and standardize a .NET 10 Blazor solution."
license: MIT
author: Raymond Andreassen
version: "1.3.0"
sourceChats:
---

- Important: DO NOT DEBUG AND CORRECT ERRORS, I AM DEBUGGING THIS SKILL AND WANT CORRECT MISTAKES
- Enable nullable reference types and treat warnings seriously.
- Add  .editorconfig  for consistent formatting and naming. TreatWarningsAsErrors sets to false.
- Set in .editorconfig
  - dotnet_diagnostic.SA1124.severity = none
  - dotnet_diagnostic.SA1516.severity = none
- Use analyzers ( StyleCop ,  Microsoft.CodeAnalysis.NetAnalyzers ).
- Do not use #region/#endregion, 
- Keep project structure simple: separate  src/  and  tests/ , one project per concern.
- Use xUnit for tests, and run them from Test Explorer.
- Prefer async/await for I/O-bound work.
- Use dependency injection and avoid hard-coded dependencies.
- Manage packages with NuGet and pin versions where possible.
- Use Git integration in Visual Studio, with small commits and pull requests.
- Use nuget to get nuget packages, always latest version, no prerelease, upgrade if newer version exist. 


# UiT MudBlazor guidance
## Required inputs (ask first)
Do not proceed past planning until all required inputs are provided.
Before making any code/config changes, collect these values from the user:

1. `applicationName`
2. `applicationVersion`, default '1.0.0'
3. `tenantId`
4. `clientId`
5. `solutionFileName`, default is applicationName with .slx file extension
6. `Solution folder`, will be path to solution + solutionFileName, do not include .slx in path
7. `Initialize git`, yes/no, use `git init` if yes. 

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
- Use Aspire, add it to the solution, add project to aspire. 

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
- Add: using Microsoft.Identity.Web.UI, in program.cs

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
- Add: using OpenTelemetry.Resources;

## Components guidance
- Scoped class for user information and UI, classname UIUser
- Singleton class for application UI, classname UIApp




## Code quality
• Prefer DI over static helpers.
• Prefer async APIs.
• Keep components small and focused.
• Reuse existing patterns in the repo before introducing new abstractions.



