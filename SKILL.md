---
name: upgrade-solution-to-dotnet-10
description: 'Upgrade an existing .NET solution to .NET 10 with a safe, repeatable workflow. Use when user asks to upgrade project/solution/framework from net6.0/net7.0/net8.0/net9.0 to net10.0, update SDK pinning, modernize package versions, and verify build/test/runtime behavior.'
---

# Upgrade Solution To .NET 10

Use this skill when a user wants to move an existing .NET solution to .NET 10.

## Goals

1. Move project target frameworks to `net10.0`.
2. Align SDK resolution (for example `global.json`) with a .NET 10 SDK.
3. Update NuGet packages and tools for compatibility.
4. Validate build, test, and runtime behavior to catch regressions.

## Preconditions

1. Confirm what should be upgraded:
   - Entire solution or selected projects.
   - Class libraries only, or app hosts too.
2. Confirm whether multi-targeting is required (for example keep `net8.0` plus `net10.0`).
3. Confirm current SDKs and pinned SDK behavior:
   - `dotnet --list-sdks`
   - `global.json` if present.

## Workflow

### 1. Create a safe baseline

1. Ensure working tree state is known (`git status`). If it is not notify user and abort.

### 2. Update target frameworks

1. Find all SDK-style projects (`*.csproj`, optionally `*.fsproj`).
2. Update either:
   - `<TargetFramework>netX.Y</TargetFramework>` to `net10.0`, or
   - `<TargetFrameworks>...</TargetFrameworks>` to include `net10.0`.
3. Keep project style and formatting unchanged.

### 4. Update package references

1. Review all `PackageReference` entries.
2. Upgrade key Microsoft stacks first (for example ASP.NET Core, EF Core, Extensions).
3. Then upgrade third-party packages to versions compatible with `net10.0`.
4. Prefer minimal, compatibility-focused updates first; avoid unrelated refactors.
5. Do not upgrade Microsoft.ApplicationInsights.AspNetCore to 3.0.0 or later unless the user explicitly requests it.

Useful commands:

```bash
dotnet list <project-or-sln> package --outdated
```

```bash
dotnet restore
```

### 5. Fix namespace collision

Fix a class namespace collision by adding these to .csproject file

```xml
	<PackageReference Include="Azure.Core" Version="1.55.0" />
	<PackageReference Include="Azure.Identity" Version="1.21.0" />
```

This should only be done for web, console and azure functions projects.

### 6. Fix compile and analyzer issues

1. Build solution and fix compile errors first.
2. Resolve API breaks from framework/package upgrades.
3. Address nullable/analyzer deltas only when they block build or are requested.

### 7. Finalize

1. Re-run full restore/build.
2. Summarize changes by category:
   - Target framework updates.
   - SDK pinning updates.
   - Package upgrades.
   - Code fixes required by breaking changes.
3. Document any deferred work (for example package held back due incompatibility).

## Guardrails

1. Do not use destructive git commands.
2. Do not silently broaden scope to unrelated cleanups.
3. Preserve existing solution structure and coding style.
4. Prefer smallest viable change set that passes build/test.

## Output Checklist

When done, report:

1. Which projects now target `net10.0`.
2. Packages changed (high-level list).
3. Build/test results and any remaining known risks.
