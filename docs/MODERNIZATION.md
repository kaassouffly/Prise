# Prise – Modernization Plan

This document outlines a comprehensive modernization strategy for the Prise plugin framework, covering framework upgrades, language improvements, dependency updates, and CI/CD pipeline enhancements.

---

## Table of Contents

- [1. Executive Summary](#1-executive-summary)
- [2. Framework Modernization](#2-framework-modernization)
- [3. C# Language Modernization](#3-c-language-modernization)
- [4. Dependency Updates](#4-dependency-updates)
- [5. CI/CD Pipeline Modernization](#5-cicd-pipeline-modernization)
- [6. Build System Modernization](#6-build-system-modernization)
- [7. Migration Guide](#7-migration-guide)
- [8. Risk Assessment](#8-risk-assessment)

---

## 1. Executive Summary

Prise currently targets .NET Core 2.1 through .NET 6.0 using C# 8–9 features. Several of these runtimes are end-of-life, and the codebase does not yet leverage modern C# capabilities. This plan proposes a phased modernization to align with current .NET LTS releases while preserving the framework's core value proposition: decoupled, backwards-compatible plugin loading.

### Current State

| Dimension | Status |
|---|---|
| Target Frameworks | netcoreapp2.1 (EOL), netcoreapp3.1 (EOL), net5.0 (EOL), net6.0 (LTS, EOL Nov 2024) |
| C# Language Version | 8–9 (mixed across projects) |
| Nullable Reference Types | Enabled only in Prise.Proxy and Prise.ReverseProxy |
| CI/CD SDK | Pinned to .NET 6.0.100 |
| Test Frameworks | MSTest 2.1.1 + xUnit 2.4.0 (mixed) |

### Target State

| Dimension | Target |
|---|---|
| Target Frameworks | net8.0 (LTS), net9.0 (STS), with netstandard2.0 for plugin contracts |
| C# Language Version | 12+ (unified) |
| Nullable Reference Types | Enabled across all projects |
| CI/CD SDK | .NET 8.0 / 9.0 with matrix builds |
| Test Frameworks | Unified on a single framework |

---

## 2. Framework Modernization

### 2.1 Phase 1: Drop End-of-Life Frameworks

**Priority:** High  
**Impact:** Reduces build matrix, simplifies conditional compilation, enables modern APIs

#### Actions

1. **Remove `netcoreapp2.1` target** from `Prise.csproj` and `Prise.Mvc.csproj`
   - End of Support: May 2021
   - Eliminates the need for `System.Text.Json` 4.6.1 workaround
   - Removes `System.Reflection.Metadata` 1.8.0 fallback
   - Removes `Microsoft.DotNet.PlatformAbstractions` dependency

2. **Remove `netcoreapp3.1` target** from `Prise.csproj` and `Prise.Mvc.csproj`
   - End of Support: December 2022
   - All conditional features (`HAS_NATIVE_RESOLVER`, `SUPPORTS_UNLOADING`, etc.) become baseline

3. **Remove `net5.0` target** from `Prise.csproj` and `Prise.Mvc.csproj`
   - End of Support: May 2022
   - `SUPPORTS_NATIVE_PLATFORM_ABSTRACTIONS` becomes baseline

#### Conditional Compilation Cleanup

Once netcoreapp2.1/3.1 are dropped, the following `#if` directives and their corresponding fallback code can be removed:

| File | Directive | Action |
|---|---|---|
| `DefaultNativeAssemblyUnloader.cs` | `#if !SUPPORTS_NATIVE_UNLOADING` | Remove fallback, keep native code |
| `InMemoryAssemblyLoadContext.cs` | `#if SUPPORTS_UNLOADING` | Remove directive, keep code |
| `DefaultAssemblyLoadContext.cs` | `#if SUPPORTS_UNLOADING` | Remove directive, keep code |
| `DefaultPluginDependencyContext.cs` | `#if SUPPORTS_NATIVE_PLATFORM_ABSTRACTIONS` | Remove directive, keep code |
| `DefaultAssemblyDependencyResolver.cs` | `#if HAS_NATIVE_RESOLVER` (4 sites) | Remove directive, keep code |
| `DefaultPluginLoader.cs` | `#if SUPPORTS_ASYNC_STREAMS` | Remove directive, keep code |
| `IPluginLoader.cs` | `#if SUPPORTS_ASYNC_STREAMS` | Remove directive, keep code |

#### Resulting Target Frameworks

```xml
<!-- Prise.csproj and Prise.Mvc.csproj -->
<TargetFrameworks>net6.0;net8.0;net9.0</TargetFrameworks>

<!-- Prise.Plugin.csproj, Prise.Proxy.csproj, Prise.ReverseProxy.csproj, Prise.Testing.csproj -->
<TargetFramework>netstandard2.0</TargetFramework>
```

### 2.2 Phase 2: Add Modern .NET Targets

**Priority:** High  
**Impact:** Supports current LTS and STS releases

#### Actions

1. **Add `net8.0` target** to Prise and Prise.Mvc
   - .NET 8 LTS (supported until November 2026)
   - Enables `FrozenDictionary`, improved `AssemblyLoadContext` APIs
   - Enables `TimeProvider` for testable time-dependent operations

2. **Add `net9.0` target** to Prise and Prise.Mvc
   - .NET 9 STS (supported until May 2026)
   - Enables latest runtime performance improvements

3. **Update dependency versions** per framework:

   ```xml
   <ItemGroup Condition="'$(TargetFramework)' == 'net8.0'">
     <PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="8.0.0" />
     <PackageReference Include="Microsoft.Extensions.DependencyModel" Version="8.0.0" />
     <PackageReference Include="Microsoft.Extensions.Http" Version="8.0.0" />
     <PackageReference Include="System.Reflection.MetadataLoadContext" Version="8.0.0" />
   </ItemGroup>
   ```

### 2.3 Phase 3: Consider netstandard2.0 → netstandard2.1 for Contract Libraries

**Priority:** Medium  
**Impact:** Enables `Span<T>`, `IAsyncDisposable`, default interface methods in plugin contracts

#### Tradeoff Analysis

| Aspect | netstandard2.0 | netstandard2.1 |
|---|---|---|
| **Compatibility** | .NET Framework 4.6.1+, all .NET Core | .NET Core 3.0+, no .NET Framework |
| **APIs** | Limited | `Span<T>`, `IAsyncDisposable`, default interface methods |
| **Plugin reach** | Broadest | Narrower but sufficient for modern stacks |

**Recommendation:** Keep `netstandard2.0` for `Prise.Plugin` (contracts) to maintain maximum plugin compatibility. Consider upgrading `Prise.Proxy` and `Prise.ReverseProxy` to `netstandard2.1` to use modern APIs internally.

---

## 3. C# Language Modernization

### 3.1 Enable Nullable Reference Types Everywhere

**Priority:** High  
**Impact:** Reduces NullReferenceException risks, improves API documentation

#### Current State

| Project | Nullable | LangVersion |
|---|---|---|
| Prise | ❌ Not enabled | 9 |
| Prise.Mvc | ❌ Not enabled | not set |
| Prise.Plugin | ❌ Not enabled | not set |
| Prise.Proxy | ✅ Enabled | 8 |
| Prise.ReverseProxy | ✅ Enabled | 8 |
| Prise.Testing | ❌ Not enabled | not set |

#### Actions

1. Add `<Nullable>enable</Nullable>` to all `.csproj` files
2. Annotate all public API parameters and return types
3. Fix resulting compiler warnings incrementally (use `#nullable disable` per-file for large files initially)
4. Key interfaces to annotate first:
   - `IPluginLoader` – return types like `Task<T>` need `T?` annotations
   - `IAssemblyScanner` – `IEnumerable<AssemblyScanResult>` result nullability
   - `IPluginActivator` – activation result nullability
   - `PluginLoadContext` – optional configuration properties

### 3.2 Adopt Modern C# Features

**Priority:** Medium  
**Impact:** Improves code readability, reduces boilerplate

#### Feature Adoption Plan

| C# Feature | Version | Applicable Areas | Example |
|---|---|---|---|
| **File-scoped namespaces** | 10 | All source files | `namespace Prise;` instead of `namespace Prise { }` |
| **Global using directives** | 10 | Common namespaces | `global using System.Threading.Tasks;` |
| **Records** | 9 | DTOs, scan results | `public record AssemblyScanResult(...)` |
| **Init-only properties** | 9 | Configuration types | `public string Path { get; init; }` |
| **Pattern matching** | 8–11 | Type checks, switch | `is not null`, list patterns |
| **Raw string literals** | 11 | Error messages, paths | `"""multi-line"""` |
| **Required members** | 11 | Configuration types | `required string PluginPath` |
| **Primary constructors** | 12 | Service classes | `class DefaultLoader(IDep dep)` |
| **Collection expressions** | 12 | Array/List creation | `[item1, item2, item3]` |

#### Recommended LangVersion Strategy

```xml
<!-- Directory.Build.props (new file) -->
<Project>
  <PropertyGroup>
    <LangVersion>12</LangVersion>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
</Project>
```

### 3.3 Add IAsyncDisposable Support

**Priority:** Medium  
**Impact:** Proper async resource cleanup for loaded assemblies

#### Current State

All disposable interfaces use `IDisposable`:
- `IPluginLoader : IDisposable`
- `IAssemblyLoader : IDisposable`
- `IPluginActivator : IDisposable`
- `IAssemblyScanner : IDisposable`

#### Proposed Change

```csharp
// Add dual support
public interface IPluginLoader : IDisposable, IAsyncDisposable
{
    // Existing sync methods...
    
    // New async disposal
    ValueTask DisposeAsync();
}
```

**Implementation:** Provide default `DisposeAsync()` implementations that call `Dispose()` for backwards compatibility, with async-optimized implementations in default classes.

---

## 4. Dependency Updates

### 4.1 NuGet Package Updates

#### Core Dependencies

| Package | Current | Target | Notes |
|---|---|---|---|
| `NuGet.Versioning` | 5.8.0 | 6.8.0+ | Major features, security fixes |
| `System.Runtime.Loader` | 4.3.0 | Consider removal | Built into net6.0+ runtime |
| `System.Reflection.Emit` | 4.7.0 | Consider removal | Built into net6.0+ runtime |
| `System.Text.Json` | 4.6.1 | 8.0.0+ | Performance, source generators |
| `Microsoft.Extensions.DependencyInjection` | 2.1.0–6.0.0 | 8.0.0 (per TFM) | Keyed services, performance |
| `Microsoft.Extensions.DependencyModel` | 2.1.0–6.0.0 | 8.0.0 (per TFM) | Bug fixes |

#### Test Dependencies

| Package | Current | Target | Notes |
|---|---|---|---|
| `Microsoft.NET.Test.Sdk` | 16.7.1 | 17.9+ | .NET 8 support |
| `MSTest.TestFramework` | 2.1.1 | 3.2+ | Modern assertions |
| `MSTest.TestAdapter` | 2.1.1 | 3.2+ | Performance |
| `Moq` | 4.14.6 | 4.20+ | .NET 8 support |
| `coverlet.collector` | 1.3.0 | 6.0+ | Better coverage reporting |

### 4.2 Remove Unnecessary Dependencies

After dropping netcoreapp2.1/3.1 targets:

| Package | Reason for Removal |
|---|---|
| `Microsoft.DotNet.PlatformAbstractions` | Only needed for netcoreapp2.1/3.1 (replaced by built-in APIs in net5.0+) |
| `System.Reflection.Metadata` 1.8.0 | Only needed for netcoreapp2.1 (replaced by MetadataLoadContext) |
| `System.Text.Json` 4.6.1 | Only needed for netcoreapp2.1 (built into net6.0+) |

### 4.3 Consider System.Text.Json Source Generators

**Priority:** Low  
**Impact:** Improved serialization performance, AOT-readiness

The proxy layer serializes parameters and results using `System.Text.Json`. Source generators can replace reflection-based serialization:

```csharp
[JsonSerializable(typeof(PluginParameter))]
[JsonSerializable(typeof(PluginResult))]
internal partial class PriseJsonContext : JsonSerializerContext { }
```

---

## 5. CI/CD Pipeline Modernization

### 5.1 Update SDK Versions

**Current:** All workflows pin `.NET SDK 6.0.100`

**Target:**

```yaml
- name: Setup .NET SDKs
  uses: actions/setup-dotnet@v4
  with:
    dotnet-version: |
      6.0.x
      8.0.x
      9.0.x
```

### 5.2 Add Build Matrix

Replace individual framework build steps with a matrix strategy:

```yaml
strategy:
  matrix:
    dotnet-version: ['6.0.x', '8.0.x', '9.0.x']
    os: [ubuntu-latest, windows-latest, macos-latest]
```

### 5.3 Add Code Coverage Reporting

```yaml
- name: Run Tests with Coverage
  run: |
    dotnet test src/Prise.Tests/Prise.Tests.csproj \
      --collect:"XPlat Code Coverage" \
      --results-directory ./coverage

- name: Upload Coverage to Codecov
  uses: codecov/codecov-action@v4
  with:
    directory: ./coverage
```

### 5.4 Add Security Scanning

```yaml
- name: Run Security Audit
  run: dotnet list package --vulnerable --include-transitive

- name: Run CodeQL Analysis
  uses: github/codeql-action/analyze@v3
```

### 5.5 Modernize Publish Workflow

- Add automated versioning via `MinVer` or `GitVersion`
- Add NuGet package validation (`dotnet package validate`)
- Add package signing
- Add release notes generation from commits

### 5.6 Add Dependency Update Automation

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "nuget"
    directory: "/src"
    schedule:
      interval: "weekly"
    groups:
      microsoft:
        patterns:
          - "Microsoft.*"
          - "System.*"
```

---

## 6. Build System Modernization

### 6.1 Add Directory.Build.props

Create a root-level `Directory.Build.props` to centralize build settings:

```xml
<Project>
  <PropertyGroup>
    <LangVersion>12</LangVersion>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
  </PropertyGroup>
  
  <PropertyGroup>
    <Authors>Maarten Merken</Authors>
    <Company>Prise</Company>
    <PackageLicenseExpression>MIT</PackageLicenseExpression>
    <RepositoryUrl>https://github.com/merken/Prise</RepositoryUrl>
    <RepositoryType>git</RepositoryType>
  </PropertyGroup>
</Project>
```

### 6.2 Add global.json

Pin SDK version for reproducible builds:

```json
{
  "sdk": {
    "version": "8.0.400",
    "rollForward": "latestPatch"
  }
}
```

### 6.3 Add Central Package Management

Use `Directory.Packages.props` to centralize NuGet versions:

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  
  <ItemGroup>
    <PackageVersion Include="NuGet.Versioning" Version="6.8.0" />
    <PackageVersion Include="Microsoft.Extensions.DependencyInjection" Version="8.0.0" />
    <!-- ... -->
  </ItemGroup>
</Project>
```

### 6.4 Add .editorconfig

Enforce consistent code style across the project:

```ini
[*.cs]
indent_style = space
indent_size = 4
dotnet_sort_system_directives_first = true
csharp_style_namespace_declarations = file_scoped:suggestion
csharp_style_var_for_built_in_types = true:suggestion
```

---

## 7. Migration Guide

### 7.1 For Plugin Authors

When Prise drops netcoreapp2.1/3.1 support:

1. **No breaking change** for plugins targeting `netstandard2.0` – these continue to work unchanged
2. Plugins targeting `netcoreapp2.1` or `netcoreapp3.1` must retarget to `net6.0+`
3. Plugin contracts (`Prise.Plugin`) remain on `netstandard2.0`

### 7.2 For Host Application Developers

1. Update host application's `.csproj` to target `net6.0` or later
2. Update `Prise` NuGet package to latest version
3. If using `Microsoft.DotNet.PlatformAbstractions`, remove the explicit reference (no longer needed)
4. All conditional compilation symbols (`HAS_NATIVE_RESOLVER`, `SUPPORTS_UNLOADING`, etc.) are always-on – no code changes needed

### 7.3 Version Compatibility Matrix

| Prise Version | Minimum .NET | Plugin Contract | Recommended .NET |
|---|---|---|---|
| 2.x (current) | .NET Core 2.1 | netstandard2.0 | .NET 6.0 |
| 3.0 (proposed) | .NET 6.0 | netstandard2.0 | .NET 8.0 |
| 4.0 (future) | .NET 8.0 | netstandard2.0 / 2.1 | .NET 9.0+ |

---

## 8. Risk Assessment

### High Risk

| Risk | Mitigation |
|---|---|
| Breaking plugin compatibility | Keep `Prise.Plugin` on `netstandard2.0`; maintain integration tests with legacy plugins |
| NuGet dependency conflicts | Use central package management; test dependency resolution across all TFMs |

### Medium Risk

| Risk | Mitigation |
|---|---|
| `IAsyncDisposable` not implemented by plugins | Provide default sync fallback implementations |
| Nullable annotations change API signatures | Use `[AllowNull]` / `[MaybeNull]` for phased adoption |
| C# 12 features require SDK 8.0+ | Gate new language features behind TFM conditions if needed |

### Low Risk

| Risk | Mitigation |
|---|---|
| CI/CD matrix build time increases | Use build caching, parallel jobs |
| `.editorconfig` style enforcement | Introduce as warnings first, then errors |
