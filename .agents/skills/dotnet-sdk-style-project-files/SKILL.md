---
name: dotnet-sdk-style-project-files
description: Designed state of SDK-style .NET project files (.csproj) - canonical section order (PROJECT, META DATA, BUILD, REFERENCES, INCLUDEs and FILEs, GENERATED FILES, SHARED PROJECTS), identity and metadata alignment (AssemblyName = PackageId = Description = file name, SourceContext assembly attribute), the canonical BUILD block (OutputPath ..\..\dist with SDK target-framework suffix, documentation file, custom compiler symbols NET_CORE;NET_8_;NET_10_;NET_10, Debug/Release deltas only, no x86/x64 duplication), project-type modules (library, contract, web, test, WinForms, hybrid WinForms + WebView2 + ASP.NET Core) and conditional feature templates (references, shared projects, T4, resx/settings, file exclusion, appsettings, embedded SPA, custom targets), plus pitfalls, validation checklist and a secondary migration algorithm. Load before creating, normalizing, reviewing or migrating any .csproj.
metadata:
  version: "2026-10-21"
---

# SDK-Style .NET Project Files

This general skill defines the designed state of SDK-style `.csproj` files. The standard is deliberately **explicit** rather than Microsoft's minimal SDK-style convention: it freezes the properties actually used in the standardization work, not every possible MSBuild default, and explains deliberate deviations rather than generic MSBuild knowledge. Migration guidance comes last (section 9).

## 1. Scope and Classification

- In scope: project identity, metadata, sections, build configuration, compiler symbols, references, generated files, T4, WinForms resources / settings, embedded SPA assets, MSBuild targets, shared imports.
- Out of scope: `Program.cs` / `Startup.cs`, initialization, DI, middleware (see `asp-net-core-development-guidelines`); source-level C# rules (see `csharp-coding-standards`); package version selection, upgrades and dependency modernization (see `package-version-preferences-dot-net`); application architecture.

Every element of a project file is classified:

| Class | Meaning |
| --- | --- |
| `STANDARD` | Always normative. |
| `CONDITIONAL STANDARD` | Normative only when the project needs the feature. "Optional" does not mean arbitrary agent choice. |
| `PROJECT-SPECIFIC` | Preserve the semantics; reposition carefully. |

Preservation rule: never remove unfamiliar targets, globs, generated-file metadata, imports, package metadata or reference types. Structural normalization is not dependency or architecture refactoring.

Examples use fictitious `Example.*` names. Never transplant real organization or project identifiers into this skill.

## 2. Canonical Section Order

1. `<Project Sdk="...">`.
2. `<!-- PROJECT -->` and the primary `<PropertyGroup>`.
3. `<!-- META DATA -->` and the metadata `<PropertyGroup>`.
4. Additional `AssemblyAttribute` `ItemGroup` where applicable.
5. `<!-- BUILD -->` common property group.
6. Configuration-specific property groups containing **only real differences**.
7. Optional build targets.
8. `<!-- REFERENCES -->`: `PackageReference`, `FrameworkReference`, `ProjectReference` and other reference constructs.
9. `<!-- INCLUDEs and FILEs -->`: includes, updates, removes, content.
10. Optional `<!-- GENERATED FILES -->`, `<!-- PROJECT STRUCTURE -->`, `<!-- SHARED PROJECTS -->`.
11. Shared `.projitems` import, usually near the end.
12. Closing `</Project>`.

Formatting: two-space XML indentation, blank lines between conceptual sections, keep comments / TODOs, keep semantic ordering where MSBuild evaluation depends on it. Do not generate meaningless empty `ItemGroup`s.

## 3. Identity and Metadata

- The project file base name (without `.csproj`) equals `AssemblyName`, `PackageId` and normally `Description`.
- `Title` may be a distinct user-facing application title.
- `Product`, `RootNamespace` and `SourceContext` are intentionally simpler than the full assembly identity, often derived from the first meaningful product / namespace token. The exact universal derivation is not settled yet; ask if ambiguous.
- Preserve existing meaningful `Company`, `Copyright`, `Authors`, `Version`, `AssemblyVersion`, `FileVersion`. Never replace them with empty values or reset versions.
- `Nullable`: if absent, set `<Nullable>disable</Nullable>` (in line with the defensive nullability of `csharp-coding-standards`). Never silently change an existing explicit value.
- `ImplicitUsings`, `ImportedNamespaces`, `Deterministic` and `Configurations` are explicit where the designed state requires them.
- `ApplicationIcon` belongs to META DATA, not PROJECT. Do not add an icon to projects without one.

### PROJECT (approved WinForms example)

```xml
<!-- PROJECT -->

<PropertyGroup>
  <OutputType>WinExe</OutputType>
  <TargetFramework>net10.0-windows</TargetFramework>
  <UseWindowsForms>true</UseWindowsForms>
  <IsPublishable>false</IsPublishable>
  <Title>Example Outpost</Title>
  <AssemblyName>Example.Applications.Outpost</AssemblyName>
  <RootNamespace>Example.Applications</RootNamespace>
  <Deterministic>false</Deterministic>
  <Nullable>disable</Nullable>
  <ImplicitUsings>enable</ImplicitUsings>
  <ImportedNamespaces></ImportedNamespaces>
  <Configurations>Debug;Release</Configurations>
  <Platforms>x64;x86</Platforms>
</PropertyGroup>
```

### META DATA (approved WinForms example)

```xml
<!-- META DATA -->

<PropertyGroup>
  <PackageId>Example.Applications.Outpost</PackageId>
  <Description>Example.Applications.Outpost</Description>
  <Product>Example</Product>
  <PackageTags></PackageTags>
  <Company></Company>
  <Copyright></Copyright>
  <Authors></Authors>
  <ApplicationIcon>application.ico</ApplicationIcon>
  <!-- VERSION -->
  <Version>1.0.3</Version>
  <AssemblyVersion>1.0.3</AssemblyVersion>
  <FileVersion>1.0.3</FileVersion>
</PropertyGroup>

<ItemGroup>
  <!-- ADDITIONAL ASSEMBLY-ATTRIBUTES -->
  <AssemblyAttribute Include="System.Reflection.AssemblyMetadataAttribute">
    <_Parameter1>SourceContext</_Parameter1>
    <_Parameter2>Example</_Parameter2>
  </AssemblyAttribute>
</ItemGroup>
```

## 4. Canonical BUILD

This is the authoritative common BUILD block for all project types, including WinForms `net10.0-windows`:

```xml
<!-- BUILD -->

<PropertyGroup>
  <OutputPath>..\..\dist</OutputPath>
  <AppendTargetFrameworkToOutputPath>true</AppendTargetFrameworkToOutputPath>
  <AppendRuntimeIdentifierToOutputPath>false</AppendRuntimeIdentifierToOutputPath>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>
  <GeneratePackageOnBuild>false</GeneratePackageOnBuild>
  <DefineConstants>$(DefineConstants);NET_CORE;NET_8_;NET_10_;NET_10</DefineConstants>
  <NoWarn>$(NoWarn);1701;1702;1591</NoWarn>
</PropertyGroup>

<PropertyGroup Condition="'$(Configuration)'=='Debug'">
  <DefineConstants>$(DefineConstants);TRACE;DEBUG</DefineConstants>
  <Optimize>False</Optimize>
</PropertyGroup>

<PropertyGroup Condition="'$(Configuration)'=='Release'">
  <DefineConstants>$(DefineConstants);TRACE</DefineConstants>
  <Optimize>True</Optimize>
</PropertyGroup>
```

Rules:

- One `OutputPath` `..\..\dist`; the SDK appends the target framework (`AppendTargetFrameworkToOutputPath=true`); `AppendRuntimeIdentifierToOutputPath=false`.
- No `BaseOutputPath` alias, no hard-coded `bin-net6.0` / `bin-net8.0` / `bin-net10.0`. Legacy `BaseOutputPath` and `DocumentationFile` are migration inputs, not substitutes.
- `GenerateDocumentationFile`, `GeneratePackageOnBuild`, common `NoWarn` and framework constants live in one unconditional group.
- Configuration-specific groups contain only the `TRACE` / `DEBUG` and `Optimize` differences - no repeated paths or warnings.
- Conditions are platform-independent when x86 / x64 behave identically. `Platforms=x64;x86` does **not** justify four identical `Debug|x64`, `Debug|x86`, `Release|x64`, `Release|x86` groups.
- `AnyCPU`-only projects may use the historical `Condition="'$(Configuration)|$(Platform)'=='Debug|AnyCPU'"` pattern; choose the condition that matches the actual semantics.
- Preserve special configurations (e.g. `Debug_Local`, `Debug 3001`) and their application-specific constants (e.g. `USE_3001`) without multiplying irrelevant platform combinations.
- Framework symbols are custom compatibility symbols, not a change of the SDK's built-in constants. Do not keep stale `NET5`, `NET8`, `NET6_0` by default. The scheme is cumulative:
  - `NET_CORE` - any .NET (Core) target, i.e. not .NET Framework;
  - `NET_<X>_` (trailing underscore) - "version X **or higher**", one symbol per version step of the ladder up to the target;
  - `NET_<X>` (no trailing underscore) - "**exactly** version X", only for the target itself.

  | Target | Symbols | Status |
  | --- | --- | --- |
  | `net10.0` / `net10.0-windows` | `NET_CORE;NET_8_;NET_10_;NET_10` | confirmed |
  | `net8.0` / `net8.0-windows` | `NET_CORE;NET_8_;NET_8` | derived from the scheme, not yet confirmed |
  | other targets (e.g. `net48`, `net6.0`, `net9.0`) | - | open; ask |

  Code uses `#if NET_8_` for "net8 or newer" and `#if NET_10` only for code that must exist exactly in net10.
- The SDK-suffixed output path differs physically from old hard-coded directories (`dist\net10.0\` instead of `dist\bin-net10.0\`). The new path stays canonical; when converting, search the repository for consumers of the old path (`.nuspec` files, build / release pipelines, scripts, versioning tooling, documentation) and adapt them in the same change, or ask if they live outside the repository.
- If parallel x86 / x64 outputs need separate directories, ask instead of changing the standard.

Further properties established in the standardization work (use where the project type needs them): `LangVersion`, `IsPackable`, `IsTestProject`, `ProduceReferenceAssembly`, `CopyLocalLockFileAssemblies`.

## 5. Project-Type Modules

| Type | SDK | Notes |
| --- | --- | --- |
| Library | `Microsoft.NET.Sdk` | Optional: `<ProduceReferenceAssembly>True</ProduceReferenceAssembly>`, `<CopyLocalLockFileAssemblies>true</CopyLocalLockFileAssemblies>`. |
| Contract | `Microsoft.NET.Sdk` | Often shared `.projitems`, WCF-related dependencies, generated resources, compatibility / legacy references. The contract role is no permission to delete unexpected references. |
| Web | `Microsoft.NET.Sdk.Web` | May have the XML documentation copy target (section 6), a `wwwroot` folder and web-specific content. |
| Test | `Microsoft.NET.Sdk` | Conditional: `<LangVersion>latest</LangVersion>`, `<IsPackable>false</IsPackable>`, `<IsTestProject>true</IsTestProject>`; optional `<Using Include="Microsoft.VisualStudio.TestTools.UnitTesting" />` in an `ItemGroup`. Versions may deliberately be `0.0.0` / `0.0.0.0`. |
| WinForms | `Microsoft.NET.Sdk` | `WinExe`, `net10.0-windows`, `UseWindowsForms=true`, optional `IsPublishable=false`, `Title`, `ApplicationIcon`, `Platforms=x64;x86`. Same canonical BUILD. PROJECT / META DATA order of section 3 is normative. |
| Hybrid WinForms + WebView2 + ASP.NET Core | `Microsoft.NET.Sdk` | Uses `<FrameworkReference Include="Microsoft.AspNetCore.App" />` and optional WebView2 / ASP.NET packages **without** switching to `Microsoft.NET.Sdk.Web`. ASP.NET dependencies alone never force an SDK change. |

## 6. Conditional Feature Templates

All paths, globs, target timings and file extensions in this section are structural examples; the real values are `PROJECT-SPECIFIC`.

### References

```xml
<!-- REFERENCES -->

<ItemGroup>
  <PackageReference Include="Example.Dependency" Version="1.2.3" />
  <PackageReference Include="Example.DesignPackage" Version="2.0.0">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
  </PackageReference>
  <FrameworkReference Include="Microsoft.AspNetCore.App" />
</ItemGroup>

<ItemGroup>
  <ProjectReference Include="..\Example.Product.Core\Example.Product.Core.csproj" />
</ItemGroup>

<ItemGroup>
  <Reference Include="Example.Legacy">
    <HintPath>..\..\vendor\Example.Legacy.dll</HintPath>
  </Reference>
</ItemGroup>
```

- Keep package identity, version, conditions and metadata unchanged.
- A net10 project referencing a `.net8.0`-named project variant is not automatically wrong. Propose a newer variant if one exists; redirect only after explicit confirmation, unless the migration task expressly authorizes it.
- TODO comments grant no permission to upgrade or replace dependencies.
- Explicit legacy `Reference` / `HintPath` entries next to packages: do not remove duplicates without investigation.

### Shared Projects

```xml
<!-- SHARED PROJECTS -->
<Import Project="..\Example.Shared\Example.Shared.projitems" Label="Shared" />
```

Imports have semantics; never delete them because the SDK's default globs include `.cs` files.

### XML Documentation from NuGet Dependencies

Only where dependency XML documentation is needed at runtime (e.g. Swagger):

```xml
<Target Name="_ResolveCopyLocalNuGetPkgXmls" AfterTargets="ResolveReferences">
  <ItemGroup>
    <!-- Copy also XML-Doc files from all PackageReferences to output dir -->
    <ReferenceCopyLocalPaths
      Include="@(ReferenceCopyLocalPaths->'%(RootDir)%(Directory)%(Filename).xml')"
      Condition="'%(ReferenceCopyLocalPaths.NuGetPackageId)'!='' and Exists('%(RootDir)%(Directory)%(Filename).xml')" />
  </ItemGroup>
</Target>
```

### T4 and Generated Files

```xml
<ItemGroup>
  <Compile Update="WebViewAdapter.cs">
    <DesignTime>True</DesignTime>
    <AutoGen>True</AutoGen>
    <DependentUpon>WebViewAdapter.tt</DependentUpon>
  </Compile>
  <None Update="WebViewAdapter.tt">
    <Generator>TextTemplatingFileGenerator</Generator>
    <LastGenOutput>WebViewAdapter.cs</LastGenOutput>
    <CustomToolNamespace>TemplateNamespace</CustomToolNamespace>
  </None>
</ItemGroup>
```

Preserve `Generator`, `LastGenOutput`, `DependentUpon`, `AutoGen`, `DesignTime`, `CustomToolNamespace`. See also `dotnet-t4-source-embedding`.

### WinForms Resources and Settings

```xml
<ItemGroup>
  <Compile Update="Properties\Resources.Designer.cs">
    <DesignTime>True</DesignTime>
    <AutoGen>True</AutoGen>
    <DependentUpon>Resources.resx</DependentUpon>
  </Compile>
  <Compile Update="Properties\Settings.Designer.cs">
    <DesignTimeSharedInput>True</DesignTimeSharedInput>
    <AutoGen>True</AutoGen>
    <DependentUpon>Settings.settings</DependentUpon>
  </Compile>
</ItemGroup>

<ItemGroup>
  <EmbeddedResource Update="Properties\Resources.resx">
    <Generator>ResXFileCodeGenerator</Generator>
    <LastGenOutput>Resources.Designer.cs</LastGenOutput>
  </EmbeddedResource>
  <None Update="Properties\Settings.settings">
    <Generator>SettingsSingleFileGenerator</Generator>
    <LastGenOutput>Settings.Designer.cs</LastGenOutput>
  </None>
</ItemGroup>
```

`PublicResXFileCodeGenerator` is a different, legitimate generator; keep the existing choice.

### Explicit File Exclusion

```xml
<ItemGroup>
  <Compile Remove="LegacyForm.cs" />
  <Compile Remove="LegacyForm.Designer.cs" />
  <EmbeddedResource Remove="LegacyForm.resx" />
  <None Remove="application.ico" />
</ItemGroup>
```

Directory level:

```xml
<ItemGroup>
  <Compile Remove="Legacy\**" />
  <Content Remove="Legacy\**" />
  <EmbeddedResource Remove="Legacy\**" />
  <None Remove="Legacy\**" />
</ItemGroup>
```

`Remove` operations are semantic; never "simplify them away".

### appsettings Copy / Publish Behavior

```xml
<ItemGroup>
  <Content Include="appsettings.json">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    <ExcludeFromSingleFile>true</ExcludeFromSingleFile>
    <CopyToPublishDirectory>PreserveNewest</CopyToPublishDirectory>
  </Content>
</ItemGroup>
```

### Embedded JavaScript / SPA Frontend

```xml
<Target Name="PreBuild" BeforeTargets="BeforeBuild" Condition="'$(Configuration)' == 'Debug'">
  <ItemGroup>
    <FrontendFiles Include="$(ProjectDir)..\..\..\frontend\dist\**\*" />
  </ItemGroup>
  <Copy SourceFiles="@(FrontendFiles)"
        DestinationFolder="$(ProjectDir)Frontend\webapp-files\%(RecursiveDir)" />
</Target>

<ItemGroup>
  <EmbeddedResource Include="Frontend\webapp-files\esm\**" />
  <EmbeddedResource Include="Frontend\webapp-files\*.js" />
  <EmbeddedResource Include="Frontend\webapp-files\*.png" />
  <EmbeddedResource Include="Frontend\webapp-files\*.html" />
</ItemGroup>

<ItemGroup>
  <EmbeddedResource Remove="Frontend\webapp-files\*.json" />
  <Compile Remove="Frontend\webapp-files\*.json" />
  <None Remove="Frontend\webapp-files\*.json" />
  <Compile Remove="Frontend\webapp-files\modules\**" />
  <EmbeddedResource Remove="Frontend\webapp-files\modules\**" />
  <None Remove="Frontend\webapp-files\modules\**" />
</ItemGroup>
```

### Folders, Service Entries and Custom Targets

```xml
<ItemGroup>
  <Folder Include="Controllers\" />
  <Folder Include="Resources\" />
  <Service Include="{00000000-0000-0000-0000-000000000000}" />
</ItemGroup>
```

The GUID is illustrative, not a required value. Preserve custom MSBuild targets completely: name, condition, `BeforeTargets` / `AfterTargets`, tasks, item transforms and paths.

## 7. Known Pitfalls

| Pitfall | Rule |
| --- | --- |
| Target framework already net10, but `OutputPath` / `DocumentationFile` still contain `bin-net6.0` / `bin-net8.0` | Apply the canonical BUILD. |
| Legacy `NET5`, `NET8`, `NET6_0` constants survived retargeting | Replace with the current symbol convention. |
| Debug and Release repeat output paths, warnings and framework constants | Keep only real deltas. |
| x86 and x64 cause a four-way Cartesian expansion | Collapse to configuration-only conditions. |
| Libraries reference older-named project variants | Cannot be inferred as wrong automatically; propose, do not rewrite. |
| Very old ASP.NET / NuGet packages in net10 projects | Out of scope here; report per `package-version-preferences-dot-net`. |
| Legacy vendor `Reference` / `HintPath` next to packages | Investigate before removing. |
| Generated resources, designer files, T4, `.settings`, `.projitems`, SPA embedding, custom targets, content copy metadata | Carry real semantics; preserve. |
| Hybrid WinForms + ASP.NET | Does not imply the Web SDK. |
| `ApplicationIcon` in PROJECT, hard-coded `bin-net10.0` output (earlier examples) | Superseded by sections 3 and 4. |

### Known Design Debt

- `AspNetCoreTemplate` (reference solution for the state of the art, see `package-version-preferences-dot-net`) does not yet follow the canonical BUILD: `OutputPath ..\..\dist\bin-net10.0\` with `AppendTargetFrameworkToOutputPath=false`, `BaseOutputPath`, symbols `NET5` / `NET8`, per-configuration duplication. Conversion deliberately postponed (decision 2026-10-21); until then this skill wins over the template for project files.

## 8. Validation Checklist

- [ ] Correct SDK and target framework; WinForms stays `Microsoft.NET.Sdk`.
- [ ] `AssemblyName` / `PackageId` / `Description` match the project file name.
- [ ] `RootNamespace` / `Product` / `SourceContext` coherent and intentionally short where appropriate.
- [ ] PROJECT and META DATA order kept; `ApplicationIcon` in META DATA.
- [ ] `Nullable` present (`disable` if it was absent); an existing value not silently changed.
- [ ] One shared output path; no hard-coded TFM directory, no `BaseOutputPath`.
- [ ] `AppendTargetFrameworkToOutputPath=true`, `AppendRuntimeIdentifierToOutputPath=false`.
- [ ] Documentation and package-generation flags explicit.
- [ ] Custom framework constants follow the cumulative scheme; special constants kept.
- [ ] Consumers of an old output path (`.nuspec`, pipelines, scripts) found and adapted or asked.
- [ ] Debug / Release contain only deltas; x86 / x64 not duplicated without need.
- [ ] Package versions and reference metadata unchanged; no automatic `ProjectReference` retargeting.
- [ ] All resource, T4, settings and designer semantics kept.
- [ ] All `Include` / `Update` / `Remove` and content metadata kept.
- [ ] Custom targets, SPA embeds, folder / service entries and `.projitems` imports kept.
- [ ] Valid XML; build / evaluation checked when tooling is available (otherwise say so); no unrelated formatting churn.
- [ ] No real organization or project identities in published examples.

## 9. Bringing Existing Project Files into the Designed State

1. Identify SDK, target framework, project type, actual configurations / platforms and project identity.
2. Classify every element: `STANDARD` / `CONDITIONAL STANDARD` / `PROJECT-SPECIFIC`.
3. Align PROJECT and META DATA fields, keeping meaningful existing metadata.
4. Introduce / normalize `SourceContext` only where the derivation is clear; ask if ambiguous.
5. Apply the canonical BUILD with the single `OutputPath`, SDK TFM suffix, shared warnings and constants.
6. Collapse identical platform / configuration groups into the actual deltas.
7. Keep package versions, package metadata, references, targets, generated resources, T4, content copy, globs and shared imports.
8. Propose uncertain `ProjectReference` variants; never rewrite silently.
9. Validate XML / MSBuild evaluation / build when tooling is available.
10. Review a minimal Git diff and report unresolved decisions.

## 10. Open Issues

1. Exact derivation of `RootNamespace`, `Product` and `SourceContext`.
2. Resolved 2026-10-21: absent `Nullable` becomes `disable` (section 3).
3. Partly resolved 2026-10-21: cumulative symbol scheme (section 4). Open: confirmation for net8, symbols for .NET Framework targets (`net48`), which versions form the ladder (why `NET_8_` but no `NET_6_` / `NET_9_`).
4. Resolved 2026-10-21: new output path stays canonical; consumers are checked and adapted (section 4).
5. Separate output directories for concurrent x86 / x64 builds.
6. Exceptions to `GenerateDocumentationFile=true` when behavior changes.

## 11. Maintaining This Skill

Maintain incrementally: keep the section structure stable, make small diffs, keep the designed state first and migration last, use generic names only. Latest explicit user corrections supersede earlier examples; when a rule changes, update rule, examples, pitfalls and checklist consistently.
