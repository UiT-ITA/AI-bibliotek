---
name: UiTBlazorTemplate-Design
description: "Blazor UiT design."
license: MIT
author: Raymond Andreassen
version: "1.0.0"
prerequesites: UiT Blazor core has been applied
sourceChats:
---

# UiT MudBlazor guidance
## Required inputs (ask first)
- none

## Rules:
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

## Code quality
- Prefer DI over static helpers.
- Prefer async APIs.
- Keep components small and focused.
- Reuse existing patterns in the repo before introducing new abstractions.
- Use analyzers ( StyleCop ,  Microsoft.CodeAnalysis.NetAnalyzers ) and consider warnings as errors.
- Keep project structure simple: separate  src/  and  tests/ , one project per concern.
- Use xUnit for tests, and run them from Test Explorer.
- Prefer async/await for I/O-bound work.
- Use dependency injection and avoid hard-coded dependencies.
- Manage packages with NuGet and pin versions where possible.
- Use Git integration in Visual Studio, with small commits and pull requests.

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

## Components guidance
- Scoped class for user information and UI, classname UIUser
- Singleton class for application UI, classname UIApp

## Remove examples
- Remove Weather.razor
- Remove Counter.razor
- Remove wwwroot\favicon.ico

## UiApp.cs example
```
```

## UiUser.cs example
```
```

## MainLayout.razor example
```
﻿
```
## NavMenu.razor example
```

```

## ToggleDarkLightMode.razor example
```
﻿
```

## LoginDisplay.razor example
```

```






