# Prise – .NET 10 SDK & C# 14 Upgrade Plan

This document provides a detailed plan for upgrading the Prise plugin framework to .NET 10 (LTS, supported until November 2028) and C# 14, covering SDK migration, new language feature adoption, runtime improvements, and Prise-specific considerations.

> **Relationship to other docs:** This document builds on the general modernization strategy in [MODERNIZATION.md](MODERNIZATION.md) and the refactoring priorities in [REFACTORING.md](REFACTORING.md). Where MODERNIZATION.md provides a broad framework upgrade roadmap, this document focuses specifically on the .NET 10 / C# 14 target with concrete code changes and migration steps.

---

## Table of Contents

- [1. Executive Summary](#1-executive-summary)
- [2. .NET 10 SDK Migration](#2-net-10-sdk-migration)
- [3. C# 14 Language Feature Adoption](#3-c-14-language-feature-adoption)
- [4. Runtime & Performance Improvements](#4-runtime--performance-improvements)
- [5. Dependency & Library Upgrades](#5-dependency--library-upgrades)
- [6. Build System & CI/CD Updates](#6-build-system--cicd-updates)
- [7. Prise-Specific Migration Considerations](#7-prise-specific-migration-considerations)
- [8. Sample & Test Project Updates](#8-sample--test-project-updates)
- [9. Migration Checklist](#9-migration-checklist)
- [10. Risk Assessment & Breaking Changes](#10-risk-assessment--breaking-changes)

---

## 1. Executive Summary

.NET 10 is a Long-Term Support (LTS) release supported until November 2028. Paired with C# 14, it introduces substantial language features (extension members, `field` keyword, partial constructors), runtime performance gains (JIT improvements, NativeAOT enhancements), and platform capabilities (AI integration, post-quantum cryptography, improved Blazor/MAUI support) that directly benefit the Prise plugin framework.

### Current State → Target State

| Dimension | Current (Prise 2.x) | Target (.NET 10) |
|---|---|---|
| **Target Frameworks** | netcoreapp2.1, 3.1, net5.0, net6.0 | **net10.0** (primary), net8.0 (secondary LTS) |
| **C# Language Version** | 8–9 (mixed) | **14** (unified) |
| **SDK Version** | 6.0.100 (CI) | **10.0.x** |
| **Nullable References** | Partial (2 of 6 projects) | **Enabled everywhere** |
| **Plugin Contracts** | netstandard2.0 | **netstandard2.0** (preserved for compatibility) |
| **NativeAOT Support** | None | **Available** (opt-in for host apps) |
| **Extension Methods** | Traditional `static class` style | **Extension members** (C# 14) |

### Why .NET 10?

1. **LTS stability** — 3 years of support (November 2025 – November 2028) for enterprise adoption
2. **Performance** — JIT inlining, devirtualization, and GC improvements directly benefit plugin proxy invocations
3. **NativeAOT** — Enables ahead-of-time compiled plugin hosts with faster startup
4. **C# 14** — Extension members and the `field` keyword solve long-standing Prise code patterns
5. **Security** — Post-quantum cryptography support future-proofs plugin signature verification

---

## 2. .NET 10 SDK Migration

### 2.1 Target Framework Updates

#### Phase 1: Drop End-of-Life Frameworks (Prerequisite)

As detailed in [MODERNIZATION.md](MODERNIZATION.md#21-phase-1-drop-end-of-life-frameworks), remove all EOL targets first:

```xml
<!-- BEFORE: Prise.csproj -->
<TargetFrameworks>netcoreapp2.1;netcoreapp3.1;net5.0;net6.0</TargetFrameworks>

<!-- AFTER: Phase 1 -->
<TargetFrameworks>net8.0;net10.0</TargetFrameworks>
```

#### Phase 2: Add net10.0 Target

**Core framework projects** (`Prise.csproj`, `Prise.Mvc.csproj`):

```xml
<TargetFrameworks>net8.0;net10.0</TargetFrameworks>
```

**Plugin contract libraries** (`Prise.Plugin.csproj`, `Prise.Proxy.csproj`, `Prise.ReverseProxy.csproj`, `Prise.Testing.csproj`):

```xml
<!-- UNCHANGED — preserve maximum plugin compatibility -->
<TargetFramework>netstandard2.0</TargetFramework>
```

> **Rationale:** Keeping plugin contracts on `netstandard2.0` ensures that plugins compiled against older .NET versions continue to work with a .NET 10 host. This is Prise's core value proposition.

#### Framework-Specific Dependencies

```xml
<!-- Prise.csproj -->
<ItemGroup Condition="'$(TargetFramework)' == 'net10.0'">
  <PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="10.0.0" />
  <PackageReference Include="Microsoft.Extensions.DependencyModel" Version="10.0.0" />
  <PackageReference Include="Microsoft.Extensions.Http" Version="10.0.0" />
  <PackageReference Include="System.Reflection.MetadataLoadContext" Version="10.0.0" />
</ItemGroup>

<ItemGroup Condition="'$(TargetFramework)' == 'net8.0'">
  <PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="8.0.0" />
  <PackageReference Include="Microsoft.Extensions.DependencyModel" Version="8.0.0" />
  <PackageReference Include="Microsoft.Extensions.Http" Version="8.0.0" />
  <PackageReference Include="System.Reflection.MetadataLoadContext" Version="8.0.0" />
</ItemGroup>
```

### 2.2 Remove Obsolete Dependencies

Packages that are built into the .NET 10 runtime and no longer need explicit references:

| Package | Reason for Removal |
|---|---|
| `System.Runtime.Loader` 4.3.0 | Built into runtime since .NET Core 1.0; explicit reference unnecessary on net8.0+ |
| `System.Reflection.Emit` 4.7.0 | Built into net8.0+ runtime |
| `System.Text.Json` 4.6.1 | Built into net6.0+ runtime; explicit low version was only for netcoreapp2.1 |
| `Microsoft.DotNet.PlatformAbstractions` | Replaced by `System.Runtime.InteropServices.RuntimeInformation` in net5.0+ |
| `System.Reflection.Metadata` 1.8.0 | Only needed for netcoreapp2.1 fallback path |

### 2.3 Conditional Compilation Cleanup

After dropping all pre-net8.0 targets, **all 9 conditional compilation blocks** become unconditionally true. This is the single largest code simplification:

| Directive | Files Affected | Action |
|---|---|---|
| `HAS_NATIVE_RESOLVER` | `DefaultAssemblyDependencyResolver.cs` (4 sites) | Remove `#if` — always true on net8.0+ |
| `SUPPORTS_UNLOADING` | `InMemoryAssemblyLoadContext.cs`, `DefaultAssemblyLoadContext.cs` | Remove `#if` — always true on net8.0+ |
| `SUPPORTS_NATIVE_UNLOADING` | `DefaultNativeAssemblyUnloader.cs` | Remove `#if` — always true on net8.0+ |
| `SUPPORTS_LOADED_ASSEMBLIES` | Assembly loading code | Remove `#if` — always true on net8.0+ |
| `SUPPORTS_ASYNC_STREAMS` | `DefaultPluginLoader.cs`, `IPluginLoader.cs` | Remove `#if` — always true on net8.0+ |
| `SUPPORTS_NATIVE_PLATFORM_ABSTRACTIONS` | `DefaultPluginDependencyContext.cs` | Remove `#if` — always true on net8.0+ |

**Result:** The `<DefineConstants>` sections in `Prise.csproj` can be completely removed, and all conditional code branches collapse to their "modern" paths.

---

## 3. C# 14 Language Feature Adoption

### 3.1 Feature-by-Feature Adoption Plan

#### `field` Keyword (Properties)

**Priority:** High  
**Impact:** Eliminates manual backing fields throughout the codebase

The `field` keyword provides direct access to the compiler-generated backing field within property accessors, removing the need for explicit backing fields when custom logic is needed.

**Applicable areas in Prise:**

```csharp
// BEFORE: Manual backing field in PluginLoadContext or similar configuration types
private string _pluginAssemblyName;
public string PluginAssemblyName
{
    get => _pluginAssemblyName;
    set => _pluginAssemblyName = value ?? throw new ArgumentNullException(nameof(value));
}

// AFTER: C# 14 field keyword
public string PluginAssemblyName
{
    get => field;
    set => field = value ?? throw new ArgumentNullException(nameof(value));
}
```

```csharp
// BEFORE: Validation in setters (e.g., plugin path configuration)
private string _pathToPlugins;
public string PathToPlugins
{
    get => _pathToPlugins;
    set
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new ArgumentException("Plugin path cannot be empty.");
        _pathToPlugins = value;
    }
}

// AFTER: C# 14 field keyword
public string PathToPlugins
{
    get => field;
    set
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new ArgumentException("Plugin path cannot be empty.");
        field = value;
    }
}
```

> **Note:** Check for existing members named `field` in any class before adopting. The `field` keyword is contextual and only applies inside property accessors, but name shadowing can cause subtle bugs.

#### Extension Members (Extension Properties, Static Extensions)

**Priority:** High  
**Impact:** Enables richer plugin configuration APIs, replaces static extension method patterns

C# 14 extends extension methods into full extension members — including extension properties, indexers, and static members.

**Applicable areas in Prise:**

```csharp
// BEFORE: Extension methods in ServiceCollectionExtensions.cs
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddPrise<T>(
        this IServiceCollection services,
        Action<PluginLoadContext> configure)
    {
        // ...
    }
}

// AFTER: C# 14 extension members (grouping related extensions)
public static class ServiceCollectionExtensions
{
    extension(IServiceCollection services)
    {
        public IServiceCollection AddPrise<T>(Action<PluginLoadContext> configure)
        {
            // 'services' is implicitly available
            // ...
        }
        
        // NEW: Extension property for checking if Prise is registered
        public bool HasPriseRegistered => services.Any(s => s.ServiceType == typeof(IPluginLoader));
    }
}
```

```csharp
// Extension properties for plugin scan results
public static class AssemblyScanResultExtensions
{
    extension(AssemblyScanResult result)
    {
        // Extension property replacing helper method
        public bool IsValid => result.AssemblyPath != null && File.Exists(result.AssemblyPath);
        
        // Extension property for quick metadata access
        public string PluginTypeName => result.PluginType?.Name ?? "Unknown";
    }
}
```

```csharp
// Static extension members for factory patterns
public static class PluginLoadContextExtensions
{
    extension(PluginLoadContext)  // Static context (no instance parameter)
    {
        public static PluginLoadContext CreateDefault(string assemblyName)
        {
            var context = new PluginLoadContext();
            context.PluginAssemblyName = assemblyName;
            // Apply defaults...
            return context;
        }
    }
}
```

#### Partial Constructors and Events

**Priority:** Medium  
**Impact:** Better source generator support, cleaner code-generation patterns in Prise.Proxy

Partial constructors allow splitting constructor logic across partial class declarations — useful for Prise's proxy generation infrastructure.

```csharp
// In Prise.Proxy — proxy types are generated dynamically
// Partial constructors enable source generators to add initialization logic

// Generated file (source generator)
public partial class GeneratedPluginProxy
{
    public partial GeneratedPluginProxy(IParameterConverter converter, IResultConverter resultConverter);
}

// Hand-written file
public partial class GeneratedPluginProxy
{
    private readonly IParameterConverter _converter;
    private readonly IResultConverter _resultConverter;
    
    public partial GeneratedPluginProxy(IParameterConverter converter, IResultConverter resultConverter)
    {
        _converter = converter ?? throw new ArgumentNullException(nameof(converter));
        _resultConverter = resultConverter ?? throw new ArgumentNullException(nameof(resultConverter));
    }
}
```

#### Unbound Generic Types in `nameof`

**Priority:** Low  
**Impact:** Cleaner diagnostic and logging messages

```csharp
// BEFORE: Must supply a type argument
var typeName = nameof(IPluginLoader);  // Works, but for generic interfaces:
var name = nameof(Task<object>);       // "Task" — but requires dummy type argument

// AFTER: C# 14 — unbound generics in nameof
var name = nameof(Task<>);             // "Task" — cleaner, no dummy type needed
var name2 = nameof(IEnumerable<>);     // "IEnumerable"
```

Useful in Prise's assembly scanning and error messages:

```csharp
// In DefaultAssemblyScanner — diagnostic messages
throw new PluginNotFoundException(
    $"No type implementing {nameof(IEnumerable<>)} of plugins found in {assemblyPath}");
```

#### Implicit Span Conversions

**Priority:** Medium  
**Impact:** Performance improvement in assembly loading and byte manipulation

```csharp
// BEFORE: Explicit conversion when loading assembly bytes
byte[] assemblyBytes = File.ReadAllBytes(assemblyPath);
ReadOnlySpan<byte> span = assemblyBytes.AsSpan();

// AFTER: C# 14 — implicit conversion
byte[] assemblyBytes = File.ReadAllBytes(assemblyPath);
ReadOnlySpan<byte> span = assemblyBytes;  // Implicit conversion
```

Applicable in `InMemoryAssemblyLoadContext` and assembly loading paths for allocation-free operations.

#### Lambda Parameter Modifiers

**Priority:** Low  
**Impact:** Cleaner lambda expressions in DI configuration and plugin activation

```csharp
// BEFORE: Must specify full parameter types for ref/out
services.AddPrise<IMyPlugin>(options =>
{
    Func<string, bool> tryParse = (string input, out bool result) => bool.TryParse(input, out result);
});

// AFTER: C# 14 — modifiers without explicit types
services.AddPrise<IMyPlugin>(options =>
{
    Func<string, bool> tryParse = (input, out result) => bool.TryParse(input, out result);
});
```

#### Null-Conditional Assignment

**Priority:** Medium  
**Impact:** Safer property assignment in plugin activation and configuration

```csharp
// BEFORE: Explicit null check before assignment
if (pluginActivationContext != null)
{
    pluginActivationContext.PluginServices = hostServices;
}

// AFTER: C# 14 null-conditional assignment
pluginActivationContext?.PluginServices = hostServices;
```

This pattern is useful throughout Prise's plugin activation pipeline where optional contexts may or may not be present.

### 3.2 Language Version Configuration

```xml
<!-- Directory.Build.props (root-level, applies to all projects) -->
<Project>
  <PropertyGroup>
    <LangVersion>14</LangVersion>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>
</Project>
```

> **Important:** `Prise.Plugin`, `Prise.Proxy`, `Prise.ReverseProxy`, and `Prise.Testing` target `netstandard2.0`. C# 14 features can still be used in these projects as long as the features don't depend on runtime APIs unavailable in netstandard2.0. The `field` keyword and null-conditional assignment are compiler features and work on all targets. Extension members may require polyfills for some APIs.

### 3.3 Feature Adoption Priority Matrix

| C# 14 Feature | Priority | Effort | Prise Impact | Applicable Projects |
|---|---|---|---|---|
| `field` keyword | 🔴 High | Low | Eliminates backing fields in config types | All |
| Extension members | 🔴 High | Medium | Richer APIs, replaces extension methods | Prise, Prise.Mvc |
| Null-conditional assignment | 🟡 Medium | Low | Safer plugin activation code | Prise, Prise.Proxy |
| Implicit Span conversions | 🟡 Medium | Low | Performance in assembly loading | Prise |
| Partial constructors | 🟡 Medium | Medium | Better proxy generation | Prise.Proxy |
| `nameof` unbound generics | 🟢 Low | Low | Cleaner diagnostics | Prise |
| Lambda parameter modifiers | 🟢 Low | Low | Minor readability gains | All |
| Compound assignment operators | 🔵 P3 | Low | Limited applicability | Prise.Proxy |

---

## 4. Runtime & Performance Improvements

### 4.1 JIT Improvements Benefiting Prise

.NET 10's JIT compiler brings improvements that directly benefit Prise's plugin proxy layer:

| JIT Improvement | Prise Benefit |
|---|---|
| **Method devirtualization** | Faster dispatch through `IPluginLoader`, `IAssemblyScanner` interfaces |
| **Improved inlining** | Plugin proxy invocations inline better, reducing call overhead |
| **Enhanced struct handling** | `AssemblyScanResult` and other value types benefit from better stack allocation |
| **AVX10.2 support** | Plugins doing numeric computation benefit from vectorized operations |

### 4.2 Garbage Collector Enhancements

.NET 10's GC improvements matter for Prise because:

- **Plugin unloading** triggers GC to collect unreferenced assemblies — improved large heap handling reduces pause times
- **Long-running services** with many loaded plugins benefit from refined memory management
- **High-throughput proxy invocations** generate less GC pressure with improved allocation patterns

### 4.3 NativeAOT Support

.NET 10 enhances NativeAOT, which is relevant for Prise host applications:

```xml
<!-- Host application .csproj — opt-in to NativeAOT -->
<PropertyGroup>
  <PublishAot>true</PublishAot>
</PropertyGroup>
```

**Considerations for Prise with NativeAOT:**

| Aspect | Status | Notes |
|---|---|---|
| Host app AOT compilation | ✅ Supported | Host can be AOT-compiled |
| Dynamic plugin loading | ⚠️ Partial | `AssemblyLoadContext` works but reflection-heavy paths need annotations |
| `DispatchProxy` | ❌ Not compatible | Uses `System.Reflection.Emit` — requires alternative for AOT |
| `MetadataLoadContext` | ✅ Supported | Assembly scanning works in AOT |
| Plugin AOT compilation | ⚠️ Experimental | Plugins can be AOT-compiled but lose dynamic loading benefits |

**Action items for NativeAOT compatibility:**
1. Add `[DynamicallyAccessedMembers]` annotations to plugin activation code
2. Investigate source-generated proxy alternatives to `DispatchProxy` for AOT scenarios
3. Document NativeAOT limitations in plugin loading

### 4.4 Security: Post-Quantum Cryptography

.NET 10 introduces post-quantum cryptographic algorithms. For the plugin signature verification feature proposed in [EXTENSION.md](EXTENSION.md#81-plugin-signature-verification):

```csharp
// Future: Post-quantum safe plugin signature verification
using System.Security.Cryptography;

public class PluginSignatureVerifier
{
    public bool VerifyPluginSignature(string assemblyPath, byte[] publicKey)
    {
        var assemblyBytes = File.ReadAllBytes(assemblyPath);
        // ML-DSA (Module-Lattice Digital Signature Algorithm) — post-quantum resistant
        using var mlDsa = MLDsa.Create(MLDsaAlgorithm.MLDsa65);
        mlDsa.ImportPublicKey(publicKey);
        return mlDsa.Verify(assemblyBytes, signature);
    }
}
```

---

## 5. Dependency & Library Upgrades

### 5.1 Package Version Matrix

| Package | Current Version | .NET 10 Version | Notes |
|---|---|---|---|
| `Microsoft.Extensions.DependencyInjection` | 2.1.0–6.0.0 | 10.0.0 | Keyed services, performance |
| `Microsoft.Extensions.DependencyModel` | 2.1.0–6.0.0 | 10.0.0 | Improved resolution |
| `Microsoft.Extensions.Http` | 3.1.0–6.0.0 | 10.0.0 | HTTP resilience, keyed services |
| `System.Reflection.MetadataLoadContext` | 4.7.0–6.0.0 | 10.0.0 | Performance improvements |
| `NuGet.Versioning` | 5.8.0 | 6.12.0+ | Latest features, security fixes |
| `System.Runtime.Loader` | 4.3.0 | **Remove** | Built into runtime |
| `System.Reflection.Emit` | 4.7.0 | **Remove** | Built into runtime |
| `System.Text.Json` | 4.6.1 | **Remove** (net10.0) / 10.0.0 (netstandard2.0) | Source generators, strict mode |
| `Microsoft.DotNet.PlatformAbstractions` | 2.1.0–3.1.0 | **Remove** | Replaced by built-in APIs |

### 5.2 Test Dependency Updates

| Package | Current | Target | Notes |
|---|---|---|---|
| `Microsoft.NET.Test.Sdk` | 16.7.1 | 17.12+ | .NET 10 support, improved discovery |
| `MSTest.TestFramework` | 2.1.1 | 3.7+ (or migrate to xUnit) | See [REFACTORING.md](REFACTORING.md#41-standardize-test-framework) |
| `MSTest.TestAdapter` | 2.1.1 | 3.7+ (or migrate to xUnit) | Same as above |
| `Moq` | 4.14.6 | 4.20+ | .NET 10 compatibility |
| `coverlet.collector` | 1.3.0 | 6.0+ | Better branch coverage |
| `xunit` | 2.4.0 | 2.9+ | Integration tests |
| `xunit.runner.visualstudio` | 2.4.0 | 2.8+ | Test runner |

### 5.3 New Optional Dependencies

| Package | Purpose | When to Add |
|---|---|---|
| `Microsoft.Extensions.AI` | Unified AI service abstractions | If adding AI-powered plugin features |
| `Microsoft.Extensions.Diagnostics` | Enhanced health checks | For plugin health monitoring (see [EXTENSION.md](EXTENSION.md#43-plugin-health-checks)) |
| `OpenTelemetry.Api` | Instrumentation | For plugin observability (see [EXTENSION.md](EXTENSION.md#51-opentelemetry-integration-priseopentelemetry)) |

---

## 6. Build System & CI/CD Updates

### 6.1 `global.json` — Pin .NET 10 SDK

```json
{
  "sdk": {
    "version": "10.0.100",
    "rollForward": "latestPatch"
  }
}
```

### 6.2 `Directory.Build.props` — Centralized Settings

```xml
<Project>
  <PropertyGroup>
    <LangVersion>14</LangVersion>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
  </PropertyGroup>

  <!-- Common metadata for NuGet packages -->
  <PropertyGroup>
    <Authors>Maarten Merken</Authors>
    <Company>Prise</Company>
    <PackageLicenseExpression>MIT</PackageLicenseExpression>
    <RepositoryUrl>https://github.com/merken/Prise</RepositoryUrl>
    <RepositoryType>git</RepositoryType>
  </PropertyGroup>
</Project>
```

### 6.3 CI/CD Workflow Updates

#### Updated `build.yml`

```yaml
name: prise-build

on:
  push:
  pull_request:

jobs:
  build:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        dotnet-version: ['8.0.x', '10.0.x']
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup .NET SDK
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ matrix.dotnet-version }}
      
      - name: Restore
        run: dotnet restore src/Prise/Prise.csproj
      
      - name: Build
        run: dotnet build src/Prise/Prise.csproj --no-restore -c Release
      
      - name: Build Mvc
        run: dotnet build src/Prise.Mvc/Prise.Mvc.csproj --no-restore -c Release
```

#### Updated `unit-tests.yml`

```yaml
name: prise-unit-tests

on:
  push:
  pull_request:

jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        dotnet-version: ['8.0.x', '10.0.x']
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup .NET SDK
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ matrix.dotnet-version }}
      
      - name: Test
        run: >
          dotnet test src/Prise.Tests/Prise.Tests.csproj
          --collect:"XPlat Code Coverage"
          --results-directory ./coverage
          -c Release
      
      - name: Upload Coverage
        if: matrix.os == 'ubuntu-latest' && matrix.dotnet-version == '10.0.x'
        uses: codecov/codecov-action@v4
        with:
          directory: ./coverage
```

#### Updated `publish-packages.yml`

```yaml
- name: Setup .NET SDK
  uses: actions/setup-dotnet@v4
  with:
    dotnet-version: '10.0.x'

- name: Pack
  run: |
    dotnet pack src/Prise/Prise.csproj -c Release -o ./nupkg
    dotnet pack src/Prise.Plugin/Prise.Plugin.csproj -c Release -o ./nupkg
    dotnet pack src/Prise.Proxy/Prise.Proxy.csproj -c Release -o ./nupkg
    dotnet pack src/Prise.ReverseProxy/Prise.ReverseProxy.csproj -c Release -o ./nupkg
    dotnet pack src/Prise.Mvc/Prise.Mvc.csproj -c Release -o ./nupkg
    dotnet pack src/Prise.Testing/Prise.Testing.csproj -c Release -o ./nupkg

- name: Validate Packages
  run: |
    dotnet tool install -g dotnet-validate
    dotnet validate package local ./nupkg/*.nupkg

- name: Push
  run: dotnet nuget push ./nupkg/*.nupkg --api-key ${{ secrets.NUGET_API_KEY }} --source https://api.nuget.org/v3/index.json
```

---

## 7. Prise-Specific Migration Considerations

### 7.1 AssemblyLoadContext Changes in .NET 10

.NET 10 includes refinements to `AssemblyLoadContext` that affect Prise's core plugin isolation:

| Change | Impact on Prise |
|---|---|
| Improved unloading reliability | `DefaultAssemblyLoadContext` and `InMemoryAssemblyLoadContext` benefit from more reliable GC-triggered unloading |
| Better diagnostic APIs | Can expose assembly load context state for plugin health monitoring |
| Performance improvements | Faster assembly resolution in load contexts with many loaded assemblies |

#### Code changes needed:

```csharp
// DefaultAssemblyLoadContext.cs — take advantage of improved APIs
public class DefaultAssemblyLoadContext : AssemblyLoadContext
{
    // .NET 10: Better constructor with explicit naming
    public DefaultAssemblyLoadContext(string name, bool isCollectible = true)
        : base(name, isCollectible)
    {
        // .NET 10: Improved diagnostics
        this.Resolving += OnResolving;
        this.ResolvingUnmanagedDll += OnResolvingUnmanagedDll;
    }
}
```

### 7.2 System.Text.Json Improvements

.NET 10 adds strict serialization modes and duplicate property handling, relevant to Prise's proxy serialization:

```csharp
// PriseProxy.cs — leverage .NET 10 JSON improvements
private static readonly JsonSerializerOptions ProxyJsonOptions = new()
{
    PropertyNameCaseInsensitive = true,
    // .NET 10: Strict mode for safer proxy parameter serialization
    AllowOutOfOrderMetadataProperties = false,
    RespectNullableAnnotations = true,    // .NET 10: Honors nullable annotations
    RespectRequiredConstructorParameters = true,  // .NET 10: Validates required params
};
```

### 7.3 Plugin Contract Evolution

Plugin contracts (`Prise.Plugin`) stay on `netstandard2.0` for maximum compatibility. However, C# 14 compiler features (not runtime features) can be used:

| Feature | Works on netstandard2.0? | Notes |
|---|---|---|
| `field` keyword | ✅ Yes | Compiler-only feature |
| Null-conditional assignment | ✅ Yes | Compiler-only feature |
| Extension members | ⚠️ Partial | Extension properties work; static extensions may need polyfills |
| `nameof` unbound generics | ✅ Yes | Compiler-only feature |
| Implicit Span conversions | ❌ No | Requires runtime `Span<T>` support (netstandard2.1+) |
| Partial constructors | ✅ Yes | Compiler-only feature |

### 7.4 Prise.Mvc: ASP.NET Core 10 Alignment

The `Prise.Mvc` package needs updates for ASP.NET Core 10:

```xml
<!-- Prise.Mvc.csproj -->
<ItemGroup Condition="'$(TargetFramework)' == 'net10.0'">
  <FrameworkReference Include="Microsoft.AspNetCore.App" />
  <PackageReference Include="Microsoft.AspNetCore.Mvc.Razor.RuntimeCompilation" Version="10.0.0" />
  <PackageReference Include="Microsoft.Extensions.FileProviders.Physical" Version="10.0.0" />
</ItemGroup>
```

ASP.NET Core 10 improvements applicable to Prise.Mvc:

- **OpenAPI 3.1 support** — Plugin-contributed endpoints can use the latest OpenAPI spec
- **Improved validation** — Better input validation for plugin controller parameters
- **Blazor improvements** — If extending Prise to Blazor (see [EXTENSION.md](EXTENSION.md#22-blazor-integration-priseblazor))

---

## 8. Sample & Test Project Updates

### 8.1 Sample Project Migration

All sample projects should be updated to net10.0:

| Sample | Current TFM | Target TFM | Notes |
|---|---|---|---|
| `Example.Console` | net6.0 | net10.0 | Straightforward update |
| `Example.Web` | net6.0 | net10.0 | Update to Minimal API style |
| `Example.Mvc.Controllers` | net6.0 | net10.0 | Update MVC references |
| `Example.Mvc.Razor` | net6.0 | net10.0 | Update Razor SDK |
| `Example.Mvc.Legacy` | netcoreapp2.1 | **Remove** | EOL, replaced by modern samples |
| `Example.Web.Legacy` | netcoreapp2.1 | **Remove** | EOL, replaced by modern samples |
| `Example.AzureFunction` | netcoreapp3.1 | net10.0 (isolated) | Migrate to isolated worker model |
| `Example.Akka` | net6.0 | net10.0 | Update Akka.NET packages |
| `Example.Avalonia` | netcoreapp3.1 | net10.0 | Update Avalonia packages |

#### New samples to add:

| Sample | Description |
|---|---|
| `Example.MinimalApi` | Minimal API with plugin endpoints (leveraging .NET 10 features) |
| `Example.SingleFile` | .NET 10 single-file app (`dotnet run script.cs`) with plugin loading |
| `Example.NativeAOT` | NativeAOT host with plugin loading (demonstrating limitations) |

### 8.2 Test Project Updates

```xml
<!-- Prise.Tests.csproj -->
<PropertyGroup>
  <TargetFrameworks>net8.0;net10.0</TargetFrameworks>
</PropertyGroup>

<ItemGroup>
  <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.12.0" />
  <!-- Consider migrating to xUnit per REFACTORING.md -->
  <PackageReference Include="MSTest.TestFramework" Version="3.7.0" />
  <PackageReference Include="MSTest.TestAdapter" Version="3.7.0" />
  <PackageReference Include="Moq" Version="4.20.72" />
  <PackageReference Include="coverlet.collector" Version="6.0.4" />
</ItemGroup>
```

### 8.3 Integration Test Plugin Updates

```xml
<!-- Integration test plugins should target net10.0 -->
<PropertyGroup>
  <TargetFramework>net10.0</TargetFramework>
</PropertyGroup>
```

Keep backwards compatibility test plugins at `netstandard2.0` and `netstandard2.1` to validate cross-version loading.

---

## 9. Migration Checklist

### Phase 1: Prerequisites (from MODERNIZATION.md)

- [ ] Drop `netcoreapp2.1` target from all projects
- [ ] Drop `netcoreapp3.1` target from all projects
- [ ] Drop `net5.0` target from all projects
- [ ] Remove all conditional compilation blocks (`#if HAS_NATIVE_RESOLVER`, etc.)
- [ ] Remove `<DefineConstants>` sections from `.csproj` files
- [ ] Remove obsolete NuGet dependencies (`PlatformAbstractions`, etc.)

### Phase 2: .NET 10 SDK Setup

- [ ] Create `global.json` with .NET 10 SDK version
- [ ] Create `Directory.Build.props` with C# 14 settings
- [ ] Update `.github/workflows/build.yml` to use .NET 10 SDK
- [ ] Update `.github/workflows/unit-tests.yml` to use .NET 10 SDK
- [ ] Update `.github/workflows/integration-tests.yml` to use .NET 10 SDK
- [ ] Update `.github/workflows/publish-packages.yml` to use .NET 10 SDK

### Phase 3: Project File Updates

- [ ] Update `Prise.csproj` — add `net10.0` target, update dependencies
- [ ] Update `Prise.Mvc.csproj` — add `net10.0` target, update ASP.NET Core references
- [ ] Update `Prise.Tests.csproj` — add `net10.0` target, update test packages
- [ ] Update `Prise.IntegrationTests.csproj` — add `net10.0` target
- [ ] Verify `Prise.Plugin.csproj` remains on `netstandard2.0`
- [ ] Verify `Prise.Proxy.csproj` remains on `netstandard2.0`
- [ ] Verify `Prise.ReverseProxy.csproj` remains on `netstandard2.0`

### Phase 4: C# 14 Language Adoption

- [ ] Enable `<LangVersion>14</LangVersion>` via `Directory.Build.props`
- [ ] Enable `<Nullable>enable</Nullable>` across all projects
- [ ] Adopt `field` keyword in property declarations
- [ ] Migrate extension methods to extension members where beneficial
- [ ] Apply null-conditional assignment (`?.=`) in plugin activation code
- [ ] Add `[DynamicallyAccessedMembers]` annotations for NativeAOT compatibility

### Phase 5: Testing & Validation

- [ ] Run unit tests on both net8.0 and net10.0
- [ ] Run integration tests with net10.0 host and netstandard2.0 plugins
- [ ] Run backwards compatibility tests with legacy plugins
- [ ] Validate NuGet package generation (`dotnet pack`)
- [ ] Test on Windows, Linux, and macOS
- [ ] Performance benchmark comparison (net8.0 vs. net10.0)

### Phase 6: Sample & Documentation Updates

- [ ] Update all sample projects to net10.0
- [ ] Remove legacy sample projects (netcoreapp2.1)
- [ ] Add new .NET 10-specific samples (MinimalApi, SingleFile, NativeAOT)
- [ ] Update README.md badges and version references
- [ ] Update docs/SUMMARY.md with new framework targets

---

## 10. Risk Assessment & Breaking Changes

### Breaking Changes

| Change | Impact | Mitigation |
|---|---|---|
| Drop netcoreapp2.1/3.1/net5.0 | ❌ Hosts on these frameworks cannot use new Prise versions | Maintain 2.x branch for bug fixes; plugin contracts unchanged |
| C# 14 `LangVersion` requirement | ⚠️ Contributors need .NET 10 SDK | `global.json` ensures correct SDK; clear contributor docs |
| `field` keyword conflicts | ⚠️ Any member named `field` becomes a keyword in property accessors | Audit codebase for `field` member names; use `@field` to escape |
| Extension members syntax | ⚠️ New syntax may confuse contributors unfamiliar with C# 14 | Document patterns; provide examples in CONTRIBUTING.md |

### Non-Breaking Benefits

| Benefit | Details |
|---|---|
| Plugin contracts stay on netstandard2.0 | ✅ All existing plugins continue to work without recompilation |
| Performance improvements are automatic | ✅ JIT/GC improvements apply to existing code without changes |
| JSON serialization improvements | ✅ Proxy layer benefits from System.Text.Json enhancements |
| NativeAOT is opt-in | ✅ No impact on existing users who don't enable AOT |

### High-Risk Areas

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| `DispatchProxy` behavior changes in .NET 10 | Low | High | Comprehensive integration tests; monitor .NET 10 release notes |
| `AssemblyLoadContext` API changes | Low | High | Unit tests covering all load/unload paths |
| Extension members in netstandard2.0 polyfill gaps | Medium | Medium | Keep traditional extension methods as fallback |
| NuGet package compatibility with older hosts | Low | Medium | Multi-target net8.0 + net10.0 in NuGet packages |
| `field` keyword naming conflicts | Low | Low | Code audit; `@field` escape syntax available |

### Rollback Strategy

If critical issues are discovered during migration:

1. **Branch strategy:** Maintain `release/2.x` branch for current stable code
2. **Dual targeting:** Keep `net8.0` as secondary target for fallback
3. **Feature flags:** Gate C# 14-specific features behind conditional compilation if needed
4. **Gradual rollout:** Release as `3.0.0-preview.1` for community testing before stable release
