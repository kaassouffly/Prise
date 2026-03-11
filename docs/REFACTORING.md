# Prise – Refactoring Plan

This document outlines a comprehensive refactoring strategy for the Prise plugin framework, covering code quality improvements, design pattern consolidation, test infrastructure unification, and architectural cleanup.

---

## Table of Contents

- [1. Executive Summary](#1-executive-summary)
- [2. Code Quality Improvements](#2-code-quality-improvements)
- [3. Architecture Refactoring](#3-architecture-refactoring)
- [4. Test Infrastructure Unification](#4-test-infrastructure-unification)
- [5. API Surface Cleanup](#5-api-surface-cleanup)
- [6. Documentation Refactoring](#6-documentation-refactoring)
- [7. Performance Optimization](#7-performance-optimization)
- [8. Implementation Priorities](#8-implementation-priorities)

---

## 1. Executive Summary

The Prise codebase has grown organically over several .NET generations, resulting in mixed patterns, duplicated code across conditional compilation blocks, and an inconsistent test infrastructure. This refactoring plan proposes targeted improvements to reduce technical debt while preserving the framework's stable public API.

### Key Metrics

| Metric | Current | Target |
|---|---|---|
| Conditional compilation sites | 9+ `#if` blocks | 0 (after framework cleanup) |
| Test frameworks | 2 (MSTest + xUnit) | 1 (unified) |
| Projects with nullable annotations | 2 of 6 | 6 of 6 |
| Obsolete public APIs | 3 attributes | 0 (remove in next major) |
| Code duplication (Proxy/ReverseProxy) | Significant | Shared base library |

---

## 2. Code Quality Improvements

### 2.1 Remove Conditional Compilation Blocks

**Priority:** High (after framework modernization)  
**Prerequisite:** Drop netcoreapp2.1 and netcoreapp3.1 targets (see [MODERNIZATION.md](MODERNIZATION.md))

Once all end-of-life frameworks are removed, the following conditional blocks become unconditionally true and can be simplified:

#### `src/Prise/AssemblyLoading/DefaultAssemblyDependencyResolver.cs`

```csharp
// BEFORE (4 conditional blocks)
#if HAS_NATIVE_RESOLVER
    private readonly AssemblyDependencyResolver resolver;
#endif

// AFTER (clean code)
private readonly AssemblyDependencyResolver resolver;
```

#### `src/Prise/DefaultPluginLoader.cs`

```csharp
// BEFORE
#if SUPPORTS_ASYNC_STREAMS
    public async IAsyncEnumerable<T> LoadPluginsAsAsyncEnumerable<T>(...)
    { ... }
#endif

// AFTER
public async IAsyncEnumerable<T> LoadPluginsAsAsyncEnumerable<T>(...)
{ ... }
```

#### `src/Prise/IPluginLoader.cs`

```csharp
// BEFORE
#if SUPPORTS_ASYNC_STREAMS
    IAsyncEnumerable<T> LoadPluginsAsAsyncEnumerable<T>(...);
#endif

// AFTER
IAsyncEnumerable<T> LoadPluginsAsAsyncEnumerable<T>(...);
```

#### Full Cleanup List

| File | Directives to Remove | Lines Affected |
|---|---|---|
| `DefaultNativeAssemblyUnloader.cs` | `SUPPORTS_NATIVE_UNLOADING` | Remove fallback branch |
| `InMemoryAssemblyLoadContext.cs` | `SUPPORTS_UNLOADING` | Remove directive only |
| `DefaultAssemblyLoadContext.cs` | `SUPPORTS_UNLOADING` | Remove directive only |
| `DefaultPluginDependencyContext.cs` | `SUPPORTS_NATIVE_PLATFORM_ABSTRACTIONS` | Remove directive and fallback |
| `DefaultAssemblyDependencyResolver.cs` | `HAS_NATIVE_RESOLVER` (×4) | Remove directives and fallbacks |
| `DefaultPluginLoader.cs` | `SUPPORTS_ASYNC_STREAMS` | Remove directive only |
| `IPluginLoader.cs` | `SUPPORTS_ASYNC_STREAMS` | Remove directive only |

### 2.2 Consolidate Prise.Proxy and Prise.ReverseProxy Shared Code

**Priority:** Medium  
**Impact:** Eliminates code duplication, simplifies maintenance

#### Current Problem

`Prise.ReverseProxy` includes compiled copies of several files from `Prise.Proxy` and `Prise` core. This means bug fixes must be applied in multiple places.

#### Proposed Solution

1. Extract shared infrastructure into a new internal package or shared source:

   ```
   src/
   ├── Prise.Proxy.Shared/           # NEW: shared source project
   │   ├── Infrastructure/
   │   │   ├── IParameterConverter.cs
   │   │   └── IResultConverter.cs
   │   └── SerializationHelpers.cs
   ├── Prise.Proxy/                   # References Shared
   └── Prise.ReverseProxy/           # References Shared
   ```

2. Alternative: Use shared source via `<Compile Include="../Shared/**/*.cs" />` to avoid an additional NuGet package.

### 2.3 Simplify DefaultPluginLoader

**Priority:** Medium  
**Impact:** Improved readability, testability

#### Current State

`DefaultPluginLoader.cs` is the largest file in the codebase, orchestrating the full plugin loading pipeline. It handles scanning, loading, activation, and proxy creation in a single class.

#### Proposed Refactoring

Extract pipeline stages into dedicated orchestrator classes:

```csharp
// BEFORE: One large class
public class DefaultPluginLoader : IPluginLoader
{
    // Scanning logic
    // Loading logic
    // Activation logic
    // Proxy creation logic
    // Caching logic
}

// AFTER: Pipeline stages as separate concerns
public class DefaultPluginLoader : IPluginLoader
{
    private readonly IPluginScanPipeline _scanPipeline;
    private readonly IPluginLoadPipeline _loadPipeline;
    private readonly IPluginActivationPipeline _activationPipeline;
    
    public DefaultPluginLoader(
        IPluginScanPipeline scanPipeline,
        IPluginLoadPipeline loadPipeline,
        IPluginActivationPipeline activationPipeline)
    {
        _scanPipeline = scanPipeline;
        _loadPipeline = loadPipeline;
        _activationPipeline = activationPipeline;
    }
}
```

### 2.4 Extract Magic Strings and Constants

**Priority:** Low  
**Impact:** Improved maintainability

Identify and centralize hardcoded values:

```csharp
// BEFORE (scattered throughout codebase)
var jsonOptions = new JsonSerializerOptions { PropertyNameCaseInsensitive = true };

// AFTER (centralized)
internal static class PriseDefaults
{
    public static readonly JsonSerializerOptions JsonOptions = new()
    {
        PropertyNameCaseInsensitive = true
    };
    
    public const string DefaultPluginPattern = "*.dll";
    public const string NativeLibraryExtensionWindows = ".dll";
    public const string NativeLibraryExtensionLinux = ".so";
    public const string NativeLibraryExtensionMacOS = ".dylib";
}
```

---

## 3. Architecture Refactoring

### 3.1 Introduce Result Types for Error Handling

**Priority:** Medium  
**Impact:** More predictable error handling, fewer exceptions in normal flow

#### Current Pattern

Plugin loading failures throw exceptions that must be caught by consumers:

```csharp
try
{
    var plugin = await loader.LoadPlugin<IMyPlugin>(scanResult);
}
catch (PluginLoadException ex) { ... }
catch (PluginActivationException ex) { ... }
```

#### Proposed Pattern

Introduce result types for recoverable failures:

```csharp
public record PluginLoadResult<T>
{
    public bool IsSuccess { get; init; }
    public T? Plugin { get; init; }
    public PluginLoadError? Error { get; init; }
    
    public static PluginLoadResult<T> Success(T plugin) => new() { IsSuccess = true, Plugin = plugin };
    public static PluginLoadResult<T> Failure(PluginLoadError error) => new() { IsSuccess = false, Error = error };
}

public record PluginLoadError(string Message, Exception? InnerException = null);

// Usage
var result = await loader.TryLoadPlugin<IMyPlugin>(scanResult);
if (result.IsSuccess)
{
    // Use result.Plugin
}
else
{
    logger.LogError("Plugin load failed: {Error}", result.Error.Message);
}
```

### 3.2 Introduce Plugin Lifecycle Events

**Priority:** Medium  
**Impact:** Better observability, debugging support

```csharp
public interface IPluginLifecycleHandler
{
    Task OnPluginDiscovered(AssemblyScanResult scanResult);
    Task OnPluginLoading(IPluginLoadContext context);
    Task OnPluginLoaded(IPluginLoadContext context, object pluginInstance);
    Task OnPluginActivated(IPluginLoadContext context, object proxyInstance);
    Task OnPluginUnloading(IPluginLoadContext context);
    Task OnPluginError(IPluginLoadContext context, Exception error);
}
```

### 3.3 Standardize Logging

**Priority:** Medium  
**Impact:** Improved debuggability, production readiness

#### Current State

Limited logging throughout the codebase. No structured logging.

#### Proposed Approach

1. Add `Microsoft.Extensions.Logging` dependency
2. Use `ILogger<T>` via dependency injection in all default implementations
3. Use `LoggerMessage.Define` source generators for high-performance logging

```csharp
public partial class DefaultPluginLoader
{
    private readonly ILogger<DefaultPluginLoader> _logger;

    [LoggerMessage(Level = LogLevel.Information, Message = "Scanning for plugins of type {PluginType} at {Path}")]
    partial void LogPluginScan(string pluginType, string path);

    [LoggerMessage(Level = LogLevel.Warning, Message = "Plugin {PluginName} failed to load: {Error}")]
    partial void LogPluginLoadFailure(string pluginName, string error);
}
```

### 3.4 Improve PluginLoadContext Builder Pattern

**Priority:** Low  
**Impact:** Better developer experience, discoverability

#### Current State

`PluginLoadContext` configuration uses a flat action delegate:

```csharp
services.AddPrise<IMyPlugin>(options => options
    .WithPluginAssemblyName("MyPlugin.dll")
    .WithDefaultOptions()
);
```

#### Proposed Enhancement

Add validation and builder pattern with clear stage separation:

```csharp
services.AddPrise<IMyPlugin>(options => options
    .FromAssembly("MyPlugin.dll")
    .DiscoverAt("./plugins")
    .WithSharedServices(services => services
        .AddSingleton<ILogger>(hostLogger)
    )
    .WithCaching(CacheStrategy.Singleton)
    .ValidateOnBuild()  // Throws at startup if misconfigured
);
```

---

## 4. Test Infrastructure Unification

### 4.1 Standardize Test Framework

**Priority:** High  
**Impact:** Consistent test patterns, simplified CI

#### Current State

| Project | Framework | Version |
|---|---|---|
| `Prise.Tests` | MSTest | 2.1.1 |
| `Prise.IntegrationTests` | xUnit | 2.4.0 |

#### Recommendation

**Standardize on xUnit** (industry standard for .NET open source):

- Better parallel test execution
- More expressive assertions with `FluentAssertions`
- Better integration with ASP.NET Core test infrastructure (`Microsoft.AspNetCore.Mvc.Testing`)
- Already used in integration tests

#### Migration Steps

1. Replace MSTest attributes in `Prise.Tests`:
   | MSTest | xUnit |
   |---|---|
   | `[TestClass]` | Remove (not needed) |
   | `[TestMethod]` | `[Fact]` |
   | `[DataRow]` | `[InlineData]` |
   | `[TestInitialize]` | Constructor |
   | `[TestCleanup]` | `IDisposable.Dispose()` |
   | `Assert.AreEqual(a, b)` | `Assert.Equal(a, b)` |
   | `Assert.IsNotNull(x)` | `Assert.NotNull(x)` |
   | `Assert.ThrowsException<T>` | `Assert.Throws<T>` |

2. Update `Prise.Tests.csproj`:
   ```xml
   <PackageReference Include="xunit" Version="2.7.0" />
   <PackageReference Include="xunit.runner.visualstudio" Version="2.5.7" />
   <PackageReference Include="Moq" Version="4.20.70" />
   <PackageReference Include="FluentAssertions" Version="6.12.0" />
   ```

### 4.2 Improve Test Coverage Structure

**Priority:** Medium  
**Impact:** Better confidence in changes, regression prevention

#### Current Gaps

1. **No multi-TFM test runs** – Unit tests only run on net6.0
2. **No property-based testing** – All tests are example-based
3. **No performance benchmarks** – No regression testing for perf

#### Proposed Test Categories

```
src/
├── Prise.Tests/                      # Unit tests (all components)
│   ├── Activation/
│   ├── AssemblyLoading/
│   ├── AssemblyScanning/
│   ├── Caching/
│   ├── Core/
│   ├── Platform/
│   ├── Proxy/
│   └── ReverseProxy/
├── Prise.Tests.Integration/          # Integration tests (real plugin loading)
├── Prise.Tests.Performance/          # NEW: BenchmarkDotNet performance tests
│   ├── PluginLoadingBenchmarks.cs
│   ├── ProxyInvocationBenchmarks.cs
│   └── AssemblyScanningBenchmarks.cs
└── Prise.Tests.Compatibility/        # NEW: Dedicated backwards compatibility tests
```

### 4.3 Add Test Helpers and Fixtures

**Priority:** Medium  
**Impact:** Reduced test boilerplate, consistent test patterns

```csharp
// Shared test fixture for plugin loading tests
public class PluginLoadingFixture : IAsyncLifetime
{
    public IPluginLoader Loader { get; private set; }
    public string PluginDirectory { get; private set; }

    public async Task InitializeAsync()
    {
        PluginDirectory = Path.Combine(Path.GetTempPath(), Guid.NewGuid().ToString());
        Directory.CreateDirectory(PluginDirectory);
        // Setup default loader with test configuration
    }

    public async Task DisposeAsync()
    {
        Loader?.Dispose();
        if (Directory.Exists(PluginDirectory))
            Directory.Delete(PluginDirectory, true);
    }
}
```

### 4.4 Add Mutation Testing

**Priority:** Low  
**Impact:** Validates test quality

Consider adding [Stryker.NET](https://stryker-mutator.io/docs/stryker-net/introduction/) for mutation testing:

```bash
dotnet tool install -g dotnet-stryker
dotnet stryker --project Prise.csproj --test-project Prise.Tests.csproj
```

---

## 5. API Surface Cleanup

### 5.1 Remove Deprecated APIs (Major Version)

**Priority:** Medium (next major version)  
**Impact:** Cleaner API, reduced confusion

#### APIs Marked Obsolete

| API | File | Obsolete Message | Action |
|---|---|---|---|
| `BootstrapperServiceAttribute.BridgeType` | `Prise.Plugin/BootstrapperServiceAttribute.cs` | "Use ProxyType instead" | Remove `BridgeType` property |
| `PluginServiceAttribute.BridgeType` | `Prise.Plugin/PluginServiceAttribute.cs` | "Use ProxyType instead" | Remove `BridgeType` property |
| `PluginFactoryAttribute` | `Prise.Plugin/PluginFactoryAttribute.cs` | "Use field injection instead" | Remove entire attribute |

#### Migration Guide for Plugin Authors

```csharp
// BEFORE (deprecated)
[PluginService(ServiceType = typeof(ILogger), BridgeType = typeof(LoggerBridge))]
public class MyPlugin { }

// AFTER (current)
[PluginService(ServiceType = typeof(ILogger), ProxyType = typeof(LoggerProxy))]
public class MyPlugin { }
```

### 5.2 Seal Internal Classes

**Priority:** Low  
**Impact:** Better encapsulation, performance

Identify classes not designed for inheritance and mark them as `sealed`:

```csharp
// Classes that should be sealed (not designed for extension)
public sealed class DefaultAssemblyScanner : IAssemblyScanner { }
public sealed class DefaultPluginActivationContext : IPluginActivationContext { }
public sealed class DefaultScopedPluginCache : IPluginCache { }
public sealed class DefaultDirectoryTraverser : IDirectoryTraverser { }
```

**Benefit:** Sealed classes enable JIT optimizations (devirtualization) and communicate design intent.

### 5.3 Standardize Naming Conventions

**Priority:** Low  
**Impact:** Consistent API surface

| Current | Proposed | Reason |
|---|---|---|
| `AssemblyScanResult` | `PluginScanResult` | More descriptive (scans for plugins, not assemblies) |
| `IAssemblyShim` | `ILoadedAssembly` | Clearer intent |
| `DefaultPluginActivationContext` | `PluginActivationContext` | Drop "Default" for non-interface implementations |

**Note:** These are breaking changes — reserve for a major version bump with `[Obsolete]` markers in the interim.

---

## 6. Documentation Refactoring

### 6.1 Add XML Documentation to Public APIs

**Priority:** High  
**Impact:** IntelliSense support, generated API documentation

#### Current State

Most public interfaces and methods lack XML documentation comments.

#### Actions

1. Enable documentation generation:
   ```xml
   <PropertyGroup>
     <GenerateDocumentationFile>true</GenerateDocumentationFile>
     <NoWarn>$(NoWarn);CS1591</NoWarn> <!-- Initially suppress, remove progressively -->
   </PropertyGroup>
   ```

2. Prioritize documentation for:
   - All public interfaces (`IPluginLoader`, `IAssemblyScanner`, etc.)
   - All public extension methods (`ServiceCollectionExtensions`)
   - All plugin attributes (`[Plugin]`, `[PluginService]`, etc.)
   - Configuration types (`PluginLoadContext`)

3. Example:
   ```csharp
   /// <summary>
   /// Entry point for discovering and loading plugins from external assemblies.
   /// </summary>
   /// <remarks>
   /// The plugin loader orchestrates the full pipeline: discovery, scanning, loading,
   /// activation, and proxy creation. Dispose the loader to unload all plugins.
   /// </remarks>
   public interface IPluginLoader : IDisposable
   {
       /// <summary>
       /// Finds a single plugin of type <typeparamref name="T"/> at the specified path.
       /// </summary>
       /// <typeparam name="T">The plugin contract interface.</typeparam>
       /// <param name="pathToPlugins">Directory to scan for plugin assemblies.</param>
       /// <returns>
       /// An <see cref="AssemblyScanResult"/> containing the discovered plugin metadata,
       /// or throws if no matching plugin is found.
       /// </returns>
       /// <exception cref="PluginNotFoundException">
       /// Thrown when no plugin implementing <typeparamref name="T"/> is found.
       /// </exception>
       Task<AssemblyScanResult> FindPlugin<T>(string pathToPlugins);
   }
   ```

### 6.2 Add Architecture Decision Records (ADRs)

**Priority:** Medium  
**Impact:** Preserves design rationale for future contributors

Create `docs/adr/` directory with decision records:

| ADR | Title | Status |
|---|---|---|
| ADR-001 | Use DispatchProxy for plugin type decoupling | Accepted |
| ADR-002 | Use AssemblyLoadContext for plugin isolation | Accepted |
| ADR-003 | Support netstandard2.0 for plugin contracts | Accepted |
| ADR-004 | JSON-based parameter serialization across proxy boundary | Accepted |
| ADR-005 | Drop netcoreapp2.1 and netcoreapp3.1 support | Proposed |

### 6.3 Add CONTRIBUTING.md

**Priority:** Medium  
**Impact:** Enables community contributions

Content:
- Development environment setup
- Build instructions
- Test execution guide
- Code style guidelines
- PR submission process
- Architecture overview for new contributors

---

## 7. Performance Optimization

### 7.1 Assembly Scanning Performance

**Priority:** Medium  
**Impact:** Faster plugin discovery in large directories

#### Current Approach

Sequential scanning of all DLLs in a directory using `MetadataLoadContext`.

#### Proposed Optimization

1. **Parallel scanning** with `Parallel.ForEachAsync` (net6.0+)
2. **Caching scan results** with file modification timestamps
3. **Filter by naming convention** before expensive metadata inspection

```csharp
// Proposed: Parallel assembly scanning
public async Task<IEnumerable<AssemblyScanResult>> ScanAsync(
    IAssemblyScannerOptions options,
    CancellationToken cancellationToken = default)
{
    var candidates = Directory.GetFiles(options.Path, "*.dll")
        .Where(f => !IsKnownFrameworkAssembly(f));  // Quick filter
    
    var results = new ConcurrentBag<AssemblyScanResult>();
    
    await Parallel.ForEachAsync(candidates, cancellationToken, async (file, ct) =>
    {
        var result = await ScanSingleAssembly(file, options, ct);
        if (result != null)
            results.Add(result);
    });
    
    return results;
}
```

### 7.2 Proxy Invocation Performance

**Priority:** Low  
**Impact:** Reduced overhead for plugin method calls

#### Current Approach

JSON serialization/deserialization for every parameter/result crossing the proxy boundary.

#### Optimization Options

1. **Short-circuit for primitive types** — Skip serialization for `int`, `string`, `bool`, etc.
2. **Object pool for serializer options** — Avoid repeated `JsonSerializerOptions` allocation
3. **Source-generated serialization** — Use `System.Text.Json` source generators for known types

### 7.3 Memory Optimization

**Priority:** Low  
**Impact:** Reduced memory footprint for loaded plugins

1. Use `ArrayPool<byte>` for assembly loading buffers
2. Implement weak references for cached plugin metadata
3. Profile memory usage with `dotnet-counters` and `dotnet-dump`

---

## 8. Implementation Priorities

### Priority Matrix

| Priority | Item | Effort | Impact | Dependencies |
|---|---|---|---|---|
| 🔴 **P0** | Remove EOL framework targets | Medium | High | None |
| 🔴 **P0** | Remove conditional compilation | Low | High | P0 framework removal |
| 🔴 **P0** | Enable nullable reference types | Medium | High | None |
| 🟡 **P1** | Unify test framework | Medium | Medium | None |
| 🟡 **P1** | Add XML documentation | High | Medium | None |
| 🟡 **P1** | Standardize logging | Medium | Medium | None |
| 🟡 **P1** | Add CONTRIBUTING.md | Low | Medium | None |
| 🟢 **P2** | Consolidate Proxy/ReverseProxy | Medium | Medium | None |
| 🟢 **P2** | Introduce result types | Medium | Medium | None |
| 🟢 **P2** | Add plugin lifecycle events | Medium | Medium | None |
| 🟢 **P2** | Add performance benchmarks | Medium | Low | None |
| 🔵 **P3** | Remove obsolete APIs | Low | Low | Major version bump |
| 🔵 **P3** | Seal internal classes | Low | Low | None |
| 🔵 **P3** | Add mutation testing | Low | Low | Test unification |
| 🔵 **P3** | Rename APIs | Low | Low | Major version bump |

### Recommended Implementation Order

```
Sprint 1: Foundation
├── Drop EOL targets (netcoreapp2.1, 3.1, net5.0)
├── Remove conditional compilation blocks
├── Enable nullable annotations (start with core interfaces)
└── Add Directory.Build.props, global.json

Sprint 2: Quality
├── Unify test framework (MSTest → xUnit)
├── Add XML documentation (public APIs)
├── Add structured logging
└── Add CONTRIBUTING.md

Sprint 3: Architecture
├── Consolidate Proxy/ReverseProxy code
├── Refactor DefaultPluginLoader (pipeline extraction)
├── Introduce result types
└── Add plugin lifecycle events

Sprint 4: Polish
├── Performance benchmarks
├── Seal internal classes
├── Remove obsolete APIs (major version)
└── Mutation testing setup
```
