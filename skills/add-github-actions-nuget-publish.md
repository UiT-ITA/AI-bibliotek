---
name: add-github-actions-nuget-publish
description: 'Add a GitHub Actions workflow to build and publish a .NET class library as a NuGet package to GitHub Packages. Uses the package version from the .csproj file, not Git tags or run numbers. Use when the user asks to add GitHub Actions, CI/CD, NuGet publishing, or GitHub Packages deployment for a class library.'
---

# Publish a Class Library to GitHub Packages

## Goals

1. Configure the selected class library's NuGet package metadata.
2. Add or update `.github/workflows/deploy-nuget.yml` at the repository root.
3. Build, pack, and publish to the repository owner's GitHub Packages NuGet feed.
4. Keep the `.csproj` file as the source of truth for the package version.
5. Update the target solution's README with the setup required to consume the published package.

This follows the build/pack/push pattern in the `UiT-ITA/UitGraphQLIknowbaseClient` repository. The reference project is at `C:\Users\eje003\source\repos\UiT-ITA\UitGraphQLIknowbaseClient`, with its library in the nested `UitGraphQLIknowbaseClient` folder. This skill is self-contained; that checkout is not required.

## 1. Inspect the Target Repository

- Read applicable repository instructions and existing workflows before editing.
- Find the repository root, GitHub owner/repository, default branch, and target `.csproj`. Use existing remotes and repository metadata rather than assuming `UiT-ITA` or `master`.
- If multiple class libraries are plausible and the user has not selected one, ask which to publish. Do not pack the entire solution.
- Inspect the project's target frameworks, `global.json`, package metadata, `Directory.Build.props`/`.targets`, NuGet configuration, and project references as relevant to packing.
- Select an SDK compatible with all target frameworks and any SDK pin. Do not copy the reference workflow's `9.0.x` for a project targeting `net10.0`; that requires a compatible .NET 10 SDK.
- Check whether restore needs private GitHub packages, whether the Actions repository has access to those packages, and whether an existing secret such as `PERSONAL_ACCESS_TOKEN` is required.
- Preserve existing workflow conventions and unrelated configuration. Update an existing publishing workflow instead of creating duplicate publishers.

Always ask the user to confirm the proposed root namespace and package ID as described below, even when both can be established from the project. Ask other questions only for information that cannot be established, such as an initial version or ambiguous release branch. Never ask the user to paste a token into chat.

## 2. Configure Package Metadata

### Propose and Confirm Package Identity Before Editing

1. Inspect an explicitly configured `RootNamespace` in the selected `.csproj` or its applicable imported properties. Use that as the proposed root namespace and proposed `PackageId`. Distinguish an explicit setting from MSBuild's implicit project-name default.
2. If no explicit `RootNamespace` is configured, inspect namespace declarations in existing hand-written classes in the target library, including file-scoped and block-scoped namespaces. Exclude generated files, `bin`/`obj`, tests, and other projects. Infer the intended library root from representative public classes and their shared namespace hierarchy; do not strip a meaningful prefix such as `UiT.` or select a subnamespace such as `.Models` merely because one class uses it.
3. If the classes do not establish a clear root, present the plausible candidates or ask the user to supply one. Do not silently use the repository or project name as a fallback.
4. Present the proposed `RootNamespace` and `PackageId`, explain where the suggestion came from, and show any existing package ID that would change. For example: "Classes use `UiT.RestClientGithub`. Suggested RootNamespace: `UiT.RestClientGithub`; PackageId: `UiT.RestClientGithub`. Confirm these values or enter your preferred values, including a different PackageId."
5. Use `vscode_askQuestions` when available, with an option to accept the suggestion and freeform input enabled so the user can edit `PackageId` independently of `RootNamespace`. Otherwise ask in chat and wait. Confirmation is mandatory before changing package metadata or generating workflow/README content based on the proposed identity. A request to invoke this skill is not itself confirmation of the inferred names.
6. If the user supplies a different package ID, use it exactly as confirmed; do not force it back to the root namespace. If the answer is ambiguous about either value, clarify before editing. A package ID and a root namespace are allowed to differ.

### Apply the Confirmed Metadata

Preserve existing versions and unrelated metadata. Explicitly set the selected `.csproj`'s `RootNamespace` and `PackageId` to the user-confirmed values, using the actual project and repository details. This example shows the default suggestion where both are the same:

```xml
<PropertyGroup>
  <IsPackable>true</IsPackable>
  <RootNamespace>UiT.ExampleClient</RootNamespace>
  <PackageId>UiT.ExampleClient</PackageId>
  <Version>1.0.0</Version>
  <Description>Project-specific package description</Description>
  <RepositoryUrl>https://github.com/OWNER/REPOSITORY</RepositoryUrl>
  <RepositoryType>git</RepositoryType>
</PropertyGroup>
```

- `UiT.ExampleClient` and `1.0.0` are examples. Use the confirmed root namespace and package ID and retain the project's existing version. Do not use the unprefixed project or repository name as a substitute for the confirmed values.
- Do not rename classes or their namespaces as part of package setup.
- Always write an explicit `<PackageId>` with the confirmed package ID. `RootNamespace` alone does not determine the NuGet package ID; without `PackageId`, packing normally falls back to the assembly/project name. For example, when the user confirms both values as `UiT.RestClientGithub`:

  ```xml
  <PropertyGroup>
    <RootNamespace>UiT.RestClientGithub</RootNamespace>
    <PackageId>UiT.RestClientGithub</PackageId>
  </PropertyGroup>
  ```

- Verify that the evaluated `RootNamespace` matches the confirmed root namespace, and that the evaluated `PackageId` and generated `.nuspec` ID match the confirmed package ID. They need not all be identical if the user chose different values. Before confirming a change to an existing package ID, explicitly tell the user that GitHub treats the corrected name as a separate package and leaves the old package in place. Do not delete the old package automatically.
- Preserve authors, company, license, readme, symbol, and other package settings. Do not invent a license.
- `<GeneratePackageOnBuild>true</GeneratePackageOnBuild>` is used by the reference project but is optional because the workflow explicitly packs. Preserve it if already present.
- Keep an explicit `<Version>` in the `.csproj`. If the project already uses `<PackageVersion>` as its package version, preserve that as the source of truth instead of adding a competing value. If both exist, `PackageVersion` must not silently override the intended version.
- Inspect evaluated MSBuild properties if imports or versioning packages may override the project version. Resolve conflicts with the user's intent without removing unrelated shared build configuration.
- Do not derive versions from tags, timestamps, commit hashes, `github.run_number`, or workflow inputs. Do not pass `--version-suffix`, `/p:Version=...`, or `/p:PackageVersion=...` in the workflow.
- The user increments the version in the `.csproj` before publishing a new package. Never bump it automatically just to make publishing succeed.

## 3. Add the Publishing Workflow

Use the following as a starting point. Replace `10.0.x` and the repository-relative project path with the discovered SDK and `.csproj`. By default, enable only manual dispatch and leave the push-to-`master` trigger commented out. For a multi-targeted project, ensure the selected SDKs can build all targets. Match maintained action versions already used by the repository; the versions below match the reference pattern.

```yaml
name: Publish NuGet Package

on:
  workflow_dispatch:
  # push:
  #   branches:
  #     - master

permissions:
  contents: read
  packages: write

env:
  CONFIGURATION: Release
  DOTNET_CORE_VERSION: 10.0.x
  PROJECT_PATH: src/ExampleClient/ExampleClient.csproj

jobs:
  build-and-publish:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_CORE_VERSION }}

      - name: Restore dependencies
        run: dotnet restore "$PROJECT_PATH"

      - name: Build
        run: dotnet build "$PROJECT_PATH" --configuration "$CONFIGURATION" --no-restore

      - name: Pack NuGet package
        run: dotnet pack "$PROJECT_PATH" --configuration "$CONFIGURATION" --no-build --output ./nupkg

      - name: Push package to GitHub Packages
        env:
          NUGET_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: >-
          dotnet nuget push "./nupkg/*.nupkg"
          --source "https://nuget.pkg.github.com/${{ github.repository_owner }}/index.json"
          --api-key "$NUGET_AUTH_TOKEN"
          --skip-duplicate
```

- Put the workflow at the repository root, not inside a nested class library directory.
- Use explicit project paths for restore, build, and pack to avoid inadvertently restoring tests or publishing other libraries.
- Default to manual-only publishing unless the user explicitly requests another trigger. Leave the commented `push`/`branches`/`master` lines in place so the user can uncomment them to enable publishing on pushes to `master`; do not enable them automatically.
- Explain how to run it on GitHub.com: open the repository's **Actions** tab, select **Publish NuGet Package**, and choose **Run workflow**. The workflow file must be present on the repository's default branch for manual dispatch to be available.
- Publishing must not run on untrusted pull requests or `pull_request_target`.
- `GITHUB_TOKEN` is provided automatically by Actions. With `packages: write`, it can publish packages associated with the repository; an existing package may need to grant the repository Actions write access in its package settings.
- `--skip-duplicate` lets repeated builds of an unchanged version succeed without replacing a published version. Publishing different contents requires a new version in the `.csproj`.
- Keep packaging sequential: restore, build, then pack with `--no-build`. Do not add a version override to any command.

## 4. Configure Private Dependency Restore Only When Needed

Public-only dependencies do not require a GitHub NuGet restore source. If private GitHub dependencies are present and a suitable authenticated source is not already configured, insert this step before restore:

```yaml
      - name: Configure GitHub Packages for restore
        env:
          NUGET_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NUGET_USERNAME: ${{ github.actor }}
        run: |
          dotnet nuget add source \
            "https://nuget.pkg.github.com/${{ github.repository_owner }}/index.json" \
            --name github \
            --username "$NUGET_USERNAME" \
            --password "$NUGET_AUTH_TOKEN" \
            --store-password-in-clear-text
```

- Inspect existing NuGet configuration first. If the source already exists, use `dotnet nuget update source` with its existing name rather than adding a duplicate. Preserve package source mappings.
- Prefer `GITHUB_TOKEN` when the dependency package grants this workflow repository Actions read access. `packages: write` includes read access, but does not itself grant access to another repository's packages.
- For cross-repository packages that cannot grant access, use a classic PAT with `read:packages` for this restore step only, stored as an Actions repository secret such as `PERSONAL_ACCESS_TOKEN` and referenced as `secrets.PERSONAL_ACCESS_TOKEN`. Instruct the user to configure that secret only when needed. Ensure the account has appropriate access to the private package, including organization SSO authorization if required.
- When using a PAT, replace `NUGET_USERNAME` with a repository variable identifying the token's account, or an already established non-secret username. Do not assume the triggering actor owns the PAT and do not copy the reference's hardcoded username.
- For a dependency under a different owner, use that dependency's actual feed URL. Publishing still targets the target repository owner's feed.
- Keep publishing authenticated with `GITHUB_TOKEN` unless an explicit repository constraint requires otherwise.
- Never embed token values in source control, print them, or read secret values into chat. Tell the user which secret or package access setting to configure themselves.
- Clear-text storage is only for the ephemeral Actions runner. Do not commit generated credentials, upload the authenticated NuGet configuration, or configure persistent local credentials during validation.

## 5. Document Consumer Setup in the Target README

Always add or update a package installation/setup section in the target solution's root `README.md`. Create the README if missing, but preserve existing content and merge with any existing NuGet instructions instead of duplicating them. Follow the README's language and style.

Adapt the following information from the reference project's README. Replace all example owners, source names, package IDs, project paths, and versions with actual target details. Do not copy the reference's dependency package ID (`UiT.IknowbaseSoapClient`) or personal username as the new package's identity.

### Consumer Authentication

- Explain that consumers need their own GitHub **classic** PAT with **`read:packages`**, created at https://github.com/settings/tokens. The token's account must have access to the package; authorize organization SSO if required.
- Distinguish this local consumer credential from Actions authentication: publishing uses the automatically provided `GITHUB_TOKEN`, while private dependency restore may require a separate read-only PAT secret. An Actions secret does not configure a developer's local NuGet client.
- Never include a real token in the README or a committed NuGet configuration. Use a masked credential prompt in examples rather than the reference's unmasked `Read-Host` prompt.

### Register or Update the NuGet Source

Include PowerShell commands for registering the actual owner's feed. For example:

```powershell
$credential = Get-Credential -UserName "YOUR-GITHUB-USERNAME" -Message "Enter your classic GitHub PAT as the password"

dotnet nuget add source "https://nuget.pkg.github.com/OWNER/index.json" `
  --name "github-OWNER" `
  --username $credential.UserName `
  --password ($credential.GetNetworkCredential().Password)
```

Also show the alternative for an already registered source, using the same credential prompt and source name:

```powershell
dotnet nuget update source "github-OWNER" `
  --source "https://nuget.pkg.github.com/OWNER/index.json" `
  --username $credential.UserName `
  --password ($credential.GetNetworkCredential().Password)
```

Explain that the user should run either add or update, not both. Mention Visual Studio's **Tools > Options > NuGet Package Manager > Package Sources** as the UI for inspecting or adding package sources; the authenticated CLI setup configures the local NuGet client used by Visual Studio too.

These commands use NuGet's default user-level configuration. Windows stores the password encrypted for the current user/machine; it cannot be copied to another machine. On non-Windows platforms, recommend a supported credential provider; do not silently add `--store-password-in-clear-text`. If documenting that fallback, explicitly explain its local clear-text storage risk and keep credentials out of the repository.

### Configure Package Source Mapping

Document the user-level Windows configuration file as `$env:APPDATA\NuGet\NuGet.Config` (normally `$HOME\AppData\Roaming\NuGet\NuGet.Config`). For other platforms, direct users to their active NuGet configuration. A repository-level `NuGet.Config` may hold shared source URLs and mappings, but never credentials.

Show a mapping fragment to merge inside the existing `<configuration>` element:

```xml
<packageSourceMapping>
  <packageSource key="github-OWNER">
    <package pattern="UiT.ExampleClient" />
  </packageSource>
  <packageSource key="nuget.org">
    <package pattern="*" />
  </packageSource>
</packageSourceMapping>
```

- Use the actual published `PackageId` as the GitHub package pattern. Include any transitive private package IDs needed by consumers, mapped to their actual feeds. Do not assume all dependencies belong to the same owner.
- Never infer this pattern from the repository name, project filename, or assembly name. For example, a project named `RestClientGithub` whose evaluated `PackageId` is `UiT.RestClientGithub` requires `<package pattern="UiT.RestClientGithub" />`, not `<package pattern="RestClientGithub" />`. Check the classes' namespaces for a potentially omitted prefix, then confirm the full ID using `dotnet msbuild <project.csproj> -getProperty:PackageId` and the generated `.nuspec` before writing the README. Use that same full ID in installation commands.
- Source keys must exactly match registered package source names, including the actual name of the nuget.org source. Ensure the nuget.org source exists and points to `https://api.nuget.org/v3/index.json`.
- Exact package IDs route the specified packages to GitHub while the wildcard allows other dependencies from nuget.org. Avoid broad owner-prefix patterns unless they are an established, intentional convention.
- Merge with existing mappings and sources. Do not create a second `<packageSourceMapping>` element or overwrite unrelated configuration.

### Install the Package

Include an installation command using the actual package ID and project-file version, with a clearly identified consumer project path:

```powershell
dotnet add ./ConsumerApp/ConsumerApp.csproj package UiT.ExampleClient --version PACKAGE_VERSION
dotnet restore ./ConsumerApp/ConsumerApp.csproj
```

Also include the equivalent XML that consumers can merge into an `<ItemGroup>` in their `.csproj` instead of running `dotnet add package`:

```xml
<ItemGroup>
  <PackageReference Include="UiT.ExampleClient" Version="PACKAGE_VERSION" />
</ItemGroup>
```

Replace `UiT.ExampleClient` and `PACKAGE_VERSION` with the full evaluated package ID and project-file package version in the generated README. Keep them identical to the CLI example and source mapping. Tell consumers to run `dotnet restore` after editing their project. If the consumer uses Central Package Management, put the version in a `<PackageVersion>` entry in `Directory.Packages.props` and omit `Version` from its `.csproj` `<PackageReference>` instead.

Explain that the version must have been published successfully before installation. Mention checking token expiry, `read:packages`, package access/SSO, feed URL, and mapping/source-name matches when restore fails with authentication or package-not-found errors.

## 6. Validate Without Publishing

1. Check workflow YAML syntax with an available YAML parser and `actionlint` if installed. Check the branch, SDK, project path, permissions, and any source names against the actual repository. Do not install tools unless needed and allowed.
2. Use the project's evaluated properties to confirm the package ID and version. Run separately so each command returns a plain value:

   ```powershell
  dotnet msbuild ./src/ExampleClient/ExampleClient.csproj -getProperty:RootNamespace
   dotnet msbuild ./src/ExampleClient/ExampleClient.csproj -getProperty:PackageId
   dotnet msbuild ./src/ExampleClient/ExampleClient.csproj -getProperty:Version
   dotnet msbuild ./src/ExampleClient/ExampleClient.csproj -getProperty:PackageVersion
   ```

3. Restore and build the selected project in Release, then pack with `--no-build` into a temporary directory outside the repository. Use the actual project path and respect any existing authentication setup. Do not put credentials in the commands.
4. Inspect the generated `.nupkg` as a ZIP and parse its `.nuspec` XML to verify the package ID and version match the evaluated project properties. Require the package ID to equal the user-confirmed package ID, including any confirmed prefix such as `UiT.`; do not require equality with `RootNamespace` if the user chose a different package ID. Verify the evaluated root namespace against its separately confirmed value. A matching README alone is not sufficient. Allow NuGet's normal version normalization, such as `1.0` becoming `1.0.0`. Do not validate only the filename.
5. Review the diff for unexpected files, version changes, secrets, or workflow version overrides. Do not modify unrelated projects.
6. Check that the README includes consumer PAT setup, add/update source commands, package source mappings, installation commands, and a `.csproj` `<PackageReference>` XML example. Confirm its feed, source keys, package IDs, and version agree with the project and workflow. Specifically compare the published package's mapping pattern, installation command ID, and `PackageReference` attributes with the evaluated `PackageId` and generated `.nuspec` metadata, including any namespace-related prefix such as `UiT.`. Parse XML examples and check PowerShell syntax without executing credential prompts or modifying local NuGet credentials.

If private dependencies or unavailable tooling prevent verification, report the exact blocker and which checks remain unverified. Do not work around access failures by removing dependencies or changing package versions.

Do not push a package, trigger a workflow, push commits, or create a release during validation unless the user explicitly requests it.

## Completion

Summarize the workflow, project, and README files changed, the release trigger, package ID/version, destination feed, and validation results. Explain that publishing uses the automatically provided `GITHUB_TOKEN`; no publishing PAT setup is required. Point to the README's consumer setup instructions for a `read:packages` token. Explain any additional private dependency restore secret or package access setting only when needed, without requesting secret values. Remind the user that future package versions are changed in the `.csproj`; existing published versions are immutable.