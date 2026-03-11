# Prise – Comprehensive Repository Summary

## Table of Contents

- [1. Project Overview](#1-project-overview)
- [2. Technologies](#2-technologies)
- [3. Repository Structure](#3-repository-structure)
- [4. Architecture](#4-architecture)
- [5. Dependencies](#5-dependencies)
- [6. Compatibility](#6-compatibility)
- [7. Implementation Details](#7-implementation-details)
- [8. Build, Test & CI/CD](#8-build-test--cicd)
- [9. Samples & Documentation](#9-samples--documentation)

---

## 1. Project Overview

**Prise** is a plugin framework for .NET (Core) applications, written in C#. It enables developers to write **decoupled, modular code** with minimal effort while maximizing customizability and backwards compatibility. Prise loads plugins from foreign assemblies, decouples local and remote dependencies, and strives to avoid assembly mismatches.

- **License:** MIT (Copyright 2019 Maarten Merken)
- **NuGet Packages:** Prise, Prise.Plugin, Prise.Proxy, Prise.ReverseProxy, Prise.Mvc, Prise.Testing

---

## 2. Technologies

| Category | Technology |
|---|---|
| **Language** | C# (LangVersion 8–9) |
| **Runtime** | .NET Core 2.1, .NET Core 3.1, .NET 5.0, .NET 6.0 |
| **Build Tool** | Cake (C# Make), dotnet CLI |
| **CI/CD** | GitHub Actions |
| **Testing** | MSTest, Moq, coverlet |
| **Serialization** | System.Text.Json |
| **DI Framework** | Microsoft.Extensions.DependencyInjection |
| **Web Frameworks** | ASP.NET Core MVC, Razor |
| **Package Manager** | NuGet |
| **Documentation** | GitHub Pages (Jekyll) |

---

## 3. Repository Structure

```
Prise/
├── src/                                # Source code
│   ├── Prise/                          # Core plugin framework
│   │   ├── Activation/                 # Plugin activation logic
│   │   ├── AssemblyLoading/            # Assembly loading strategies
│   │   ├── AssemblyScanning/           # Assembly discovery and scanning
│   │   ├── Caching/                    # Plugin caching
│   │   ├── Core/                       # Core types (PluginLoadContext, etc.)
│   │   ├── DependencyInjection/        # DI integration extensions
│   │   ├── Infrastructure/             # Converters and utilities
│   │   ├── Platform/                   # Platform-specific implementations
│   │   ├── Utils/                      # Utility functions
│   │   ├── DefaultPluginLoader.cs      # Main plugin loader implementation
│   │   └── IPluginLoader.cs            # Primary loader interface
│   │
│   ├── Prise.Plugin/                   # Plugin contract library (netstandard2.0)
│   │   ├── IPluginBootstrapper.cs      # Plugin bootstrapper interface
│   │   ├── PluginAttribute.cs          # [Plugin] attribute
│   │   ├── PluginBootstrapperAttribute.cs
│   │   ├── PluginServiceAttribute.cs
│   │   ├── BootstrapperServiceAttribute.cs
│   │   ├── PluginFactoryAttribute.cs
│   │   └── PluginActivatedAttribute.cs
│   │
│   ├── Prise.Proxy/                    # Proxy generation for type decoupling
│   │   ├── runtime/                    # DispatchProxy runtime (IL generation)
│   │   ├── Infrastructure/             # Parameter/Result converters
│   │   ├── PriseProxy.cs               # Core proxy invocation logic
│   │   └── ProxyCreator.cs             # Proxy instance factory
│   │
│   ├── Prise.ReverseProxy/             # Reverse proxy (host→plugin communication)
│   │
│   ├── Prise.Mvc/                      # ASP.NET Core MVC integration
│   │
│   ├── Prise.Testing/                  # Testing utilities for plugin developers
│   │
│   ├── Prise.Tests/                    # Unit tests
│   │   ├── Activation/
│   │   ├── AssemblyLoading/
│   │   ├── AssemblyScanning/
│   │   ├── Caching/
│   │   ├── Core/
│   │   ├── Platform/
│   │   ├── Proxy/
│   │   └── ReverseProxy/
│   │
│   └── Prise.Tests.Integration/        # Integration tests
│       ├── IntegrationTestsPlugins/     # Test plugins (A, B, C)
│       ├── Prise.IntegrationTests/
│       ├── Prise.IntegrationTestsContract/
│       └── Prise.IntegrationTestsHost/
│
├── samples/                            # Example applications
│   ├── Example.Console/                # Console app with plugins
│   ├── Example.Web/                    # ASP.NET Core Web API
│   ├── Example.Mvc.Controllers/        # MVC with plugin controllers
│   ├── Example.Mvc.Razor/              # MVC with plugin Razor views
│   ├── Example.Mvc.Legacy/             # Legacy MVC example
│   ├── Example.AzureFunction/          # Azure Functions host
│   ├── Example.Akka/                   # Akka.NET integration
│   ├── Example.Avalonia/               # Avalonia cross-platform UI
│   └── Plugins/                        # Sample plugins
│       ├── Plugin.FileSystem/
│       ├── Plugin.Sql/
│       ├── Plugin.SqlClient/
│       ├── Plugin.AzureTableStorage/
│       ├── Plugin.FromHttpBody/
│       ├── MvcPlugin.DataStorage/
│       └── MvcPlugin.Twitter/
│
├── docs/                               # Documentation (GitHub Pages)
├── .github/workflows/                  # CI/CD pipelines
├── README.md
├── LICENSE
└── _config.yml                         # Jekyll config for docs site
```

---

## 4. Architecture

### Plugin Loading Pipeline

The framework follows a modular pipeline-based architecture:

```
Discovery → Scanning → Loading → Activation → Proxy Creation
```

1. **Discovery** – Locate plugin assemblies on disk via configurable directory traversal.
2. **Scanning** – Inspect assemblies using reflection metadata to find types decorated with `[Plugin]`.
3. **Loading** – Load the assembly into an isolated `AssemblyLoadContext`, resolving dependencies.
4. **Activation** – Instantiate the plugin, inject services, and invoke bootstrappers.
5. **Proxy Creation** – Wrap the plugin behind a `DispatchProxy` to decouple host and plugin type systems.

### Core Components

| Component | Responsibility |
|---|---|
| `IPluginLoader` / `DefaultPluginLoader` | Orchestrates the full load pipeline |
| `IAssemblyScanner` | Discovers plugin types in assemblies |
| `IAssemblyLoader` / `IAssemblyLoadContext` | Loads assemblies into isolated contexts |
| `IPluginDependencyResolver` | Resolves plugin dependencies |
| `IPluginActivator` | Instantiates and configures plugin instances |
| `IPluginCache` | Caches loaded plugin assemblies |
| `PriseProxy` / `ProxyCreator` | Creates DispatchProxy instances for type decoupling |
| `IParameterConverter` / `IResultConverter` | Serializes parameters/results across the proxy boundary |
| `IRuntimePlatformContext` | Provides platform-specific runtime information |

### Design Patterns

| Pattern | Usage |
|---|---|
| **Proxy** | `DispatchProxy` for decoupling host and plugin type systems |
| **Strategy** | Pluggable resolvers, loaders, converters, and scanners |
| **Factory** | Plugin activation factories and proxy creators |
| **Dependency Injection** | Full integration with `Microsoft.Extensions.DependencyInjection` |
| **Builder** | Fluent `PluginLoadContext` configuration |
| **Template Method** | Loader pipeline stages |
| **Adapter** | Parameter/Result converters for cross-boundary serialization |

### Assembly Isolation

Prise uses .NET's `AssemblyLoadContext` to create isolated environments for plugins. Each plugin gets its own load context, which:

- Prevents dependency version conflicts between host and plugins
- Supports unloading of plugins (on .NET Core 3.1+)
- Handles native library loading (on .NET Core 3.1+)
- Falls back to host dependencies when appropriate

---

## 5. Dependencies

### Core Package Dependencies

#### Prise (Core Framework)

| Package | Version (netcoreapp2.1) | Version (netcoreapp3.1) | Version (net5.0) | Version (net6.0) |
|---|---|---|---|---|
| Microsoft.Extensions.DependencyInjection | 2.1.0 | 3.1.0 | 5.0.0 | 6.0.0 |
| Microsoft.Extensions.DependencyModel | 2.1.0 | 3.1.0 | 5.0.0 | 6.0.0 |
| Microsoft.Extensions.Http | — | 3.1.0 | 5.0.0 | 6.0.0 |
| System.Reflection.MetadataLoadContext | — | 4.7.0 | 5.0.0 | 6.0.0 |
| System.Reflection.Metadata | 1.8.0 | — | — | — |
| System.Text.Json | 4.6.1 | — | — | — |
| Microsoft.DotNet.PlatformAbstractions | 2.1.0 | 3.1.0 | — | — |
| NuGet.Versioning | 5.8.0 | 5.8.0 | 5.8.0 | 5.8.0 |
| System.Runtime.Loader | 4.3.0 | 4.3.0 | 4.3.0 | 4.3.0 |

#### Prise.Plugin

| Package | Version |
|---|---|
| Microsoft.Extensions.DependencyInjection | 2.1.0 |

#### Prise.Proxy

| Package | Version |
|---|---|
| System.Reflection.Emit | 4.7.0 |

#### Prise.ReverseProxy

| Package | Version |
|---|---|
| System.Text.Json | 4.6.1 |
| System.Reflection.Emit | 4.7.0 |

#### Prise.Mvc

| Package | Versions (framework-dependent) |
|---|---|
| Microsoft.AspNetCore.Mvc.Core | 2.2.5 |
| Microsoft.AspNetCore.Mvc.Razor | 2.2.0 |
| Microsoft.AspNetCore.Mvc.Razor.RuntimeCompilation | 3.0.2–6.0.0 |
| Microsoft.Extensions.FileProviders.Physical | 2.1.0–6.0.0 |
| Microsoft.Extensions.DependencyInjection.Abstractions | 3.1.0–6.0.0 |

#### Test Dependencies

| Package | Version |
|---|---|
| Microsoft.NET.Test.Sdk | 16.7.1 |
| MSTest.TestAdapter | 2.1.1 |
| MSTest.TestFramework | 2.1.1 |
| Moq | 4.14.6 |
| coverlet.collector | 1.3.0 |

### Project References

| Project | References |
|---|---|
| Prise | Prise.Plugin, Prise.Proxy |
| Prise.Mvc | Prise, Prise.Plugin |
| Prise.ReverseProxy | Prise.Plugin |
| Prise.Testing | Prise.Plugin |
| Prise.Tests | Prise, Prise.Proxy, Prise.ReverseProxy |

---

## 6. Compatibility

### Target Frameworks

| Project | Target Frameworks |
|---|---|
| **Prise** | netcoreapp2.1, netcoreapp3.1, net5.0, net6.0 |
| **Prise.Mvc** | netcoreapp2.1, netcoreapp3.1, net5.0, net6.0 |
| **Prise.Plugin** | netstandard2.0 |
| **Prise.Proxy** | netstandard2.0 |
| **Prise.ReverseProxy** | netstandard2.0 |
| **Prise.Testing** | netstandard2.0 |
| **Prise.Tests** | net6.0 |

### Conditional Compilation Features

Prise uses `DefineConstants` to enable framework-specific features:

| Feature Flag | Available From | Description |
|---|---|---|
| `HAS_NATIVE_RESOLVER` | netcoreapp3.1+ | Native assembly resolution support |
| `SUPPORTS_UNLOADING` | netcoreapp3.1+ | Plugin assembly unloading |
| `SUPPORTS_NATIVE_UNLOADING` | netcoreapp3.1+ | Native library unloading |
| `SUPPORTS_LOADED_ASSEMBLIES` | netcoreapp3.1+ | Access to loaded assemblies list |
| `SUPPORTS_ASYNC_STREAMS` | netcoreapp3.1+ | `IAsyncEnumerable<T>` support |
| `SUPPORTS_NATIVE_PLATFORM_ABSTRACTIONS` | net5.0+ | Native platform abstractions without external package |

### Platform Support

- **Windows, Linux, macOS** – via .NET Core cross-platform runtime
- **Azure Functions** – demonstrated in `Example.AzureFunction`
- **ASP.NET Core** – Web API, MVC Controllers, Razor Views
- **Console Applications** – demonstrated in `Example.Console`
- **Akka.NET** – demonstrated in `Example.Akka`
- **Avalonia (Cross-Platform UI)** – demonstrated in `Example.Avalonia`

### Backwards Compatibility

A dedicated CI workflow (`integration-tests-backwards-compatibility.yml`) verifies that plugins built on older .NET versions (netstandard2.0) continue to work with newer host applications.

---

## 7. Implementation Details

### Key Interfaces

| Interface | Location | Purpose |
|---|---|---|
| `IPluginLoader` | `src/Prise/IPluginLoader.cs` | Entry point for loading plugins |
| `IAssemblyScanner` | `src/Prise/AssemblyScanning/` | Discovers plugin types |
| `IAssemblyLoader` | `src/Prise/AssemblyLoading/` | Loads assemblies |
| `IAssemblyLoadContext` | `src/Prise/AssemblyLoading/` | Manages assembly load contexts |
| `IPluginDependencyResolver` | `src/Prise/AssemblyLoading/` | Resolves plugin dependencies |
| `IPluginActivator` | `src/Prise/Activation/` | Activates plugin instances |
| `IPluginCache` | `src/Prise/Caching/` | Caches loaded plugin assemblies |
| `IPluginBootstrapper` | `src/Prise.Plugin/` | Plugin initialization hook |
| `IParameterConverter` | `src/Prise.Proxy/Infrastructure/` | Converts parameters across proxy boundary |
| `IResultConverter` | `src/Prise.Proxy/Infrastructure/` | Converts results across proxy boundary |
| `IRuntimePlatformContext` | `src/Prise/Platform/` | Platform-specific runtime information |

### Plugin Attributes

| Attribute | Purpose |
|---|---|
| `[Plugin]` | Marks a class as a plugin implementation |
| `[PluginBootstrapper]` | Marks a class as a plugin bootstrapper |
| `[PluginService]` | Marks a service dependency for injection into plugins |
| `[BootstrapperService]` | Marks a service for bootstrapper injection |
| `[PluginFactory]` | Marks a factory method *(deprecated)* |
| `[PluginActivated]` | Marks a method to call after plugin activation |

### Key Classes

| Class | Location | Purpose |
|---|---|---|
| `DefaultPluginLoader` | `src/Prise/DefaultPluginLoader.cs` | Main loader orchestrating the pipeline |
| `PluginLoadContext` | `src/Prise/Core/PluginLoadContext.cs` | Configuration for loading a plugin |
| `DefaultAssemblyLoadContext` | `src/Prise/AssemblyLoading/` | Isolated assembly context implementation |
| `InMemoryAssemblyLoadContext` | `src/Prise/AssemblyLoading/` | In-memory assembly loading |
| `PriseProxy` | `src/Prise.Proxy/PriseProxy.cs` | Proxy invocation handler |
| `ProxyCreator` | `src/Prise.Proxy/ProxyCreator.cs` | Creates proxy instances |
| `DefaultPluginActivationContext` | `src/Prise/Activation/` | Context for plugin activation |
| `DefaultScopedPluginCache` | `src/Prise/Caching/` | Scoped caching of loaded plugins |
| `DefaultDirectoryTraverser` | `src/Prise/AssemblyScanning/` | Filesystem traversal for plugin discovery |

### Proxy-Based Type Decoupling

Prise uses **dynamic proxy generation** (`DispatchProxy` pattern) to decouple host and plugin type systems:

1. The host defines a contract interface (e.g., `IWeatherPlugin`).
2. The plugin implements the contract, but compiles against its own version of the contract assembly.
3. At runtime, Prise creates a `DispatchProxy` that implements the host's contract interface.
4. Method calls on the proxy are intercepted and forwarded to the plugin instance.
5. Parameters and return values are serialized/deserialized (via JSON) to cross the type boundary.

This approach allows the host and plugin to use **different versions** of shared dependencies without conflicts.

### Dependency Injection Integration

Prise integrates with `Microsoft.Extensions.DependencyInjection` through extension methods in `ServiceCollectionExtensions`:

```csharp
services.AddPrise<IMyPlugin>(options => options
    .WithPluginAssemblyName("MyPlugin.dll")
    .WithDefaultOptions()
);
```

---

## 8. Build, Test & CI/CD

### Build System

- **Primary:** `dotnet build` via CLI
- **Automation:** Cake build script (`src/build.cake`)
- **Targets:** Builds all projects for all target frameworks, generates NuGet packages

### Test Suite

| Test Type | Project | Framework | Runner |
|---|---|---|---|
| **Unit Tests** | `src/Prise.Tests/` | net6.0 | MSTest |
| **Integration Tests** | `src/Prise.Tests.Integration/` | net6.0 | MSTest |
| **Backwards Compatibility** | Integration tests with net2.0 plugins | net6.0 | MSTest |
| **Sample Tests** | Sample plugin unit tests | varies | MSTest |

**Run unit tests:**
```bash
dotnet test src/Prise.Tests/Prise.Tests.csproj -f net6.0
```

### GitHub Actions Workflows

| Workflow | File | Trigger | Purpose |
|---|---|---|---|
| Build | `build.yml` | push, PR | Builds all projects |
| Unit Tests | `unit-tests.yml` | push, PR | Runs unit tests |
| Integration Tests | `integration-tests.yml` | push, PR | Tests real plugin loading |
| Backwards Compatibility | `integration-tests-backwards-compatibility.yml` | push, PR | Tests old plugin compatibility |
| Build Samples | `build-samples.yaml` | push, PR | Builds example apps |
| Build Sample Plugins | `build-samples-plugins.yaml` | push, PR | Builds example plugins |
| Sample Unit Tests | `unit-test-samples-plugins.yaml` | push, PR | Tests example plugins |
| Publish Packages | `publish-packages.yml` | manual | Publishes to NuGet |

### NuGet Packages

| Package | Current Version |
|---|---|
| Prise | 2.0.1 |
| Prise.Plugin | 2.0.0 |
| Prise.Proxy | 2.0.0 |
| Prise.ReverseProxy | 2.0.0 |
| Prise.Mvc | 2.0.0 |
| Prise.Testing | 2.0.0 |

---

## 9. Samples & Documentation

### Example Applications

| Sample | Description |
|---|---|
| `Example.Console` | Console application loading plugins dynamically |
| `Example.Web` | ASP.NET Core Web API with plugin-based services |
| `Example.Mvc.Controllers` | MVC application with plugin controllers |
| `Example.Mvc.Razor` | MVC application with plugin Razor views |
| `Example.Mvc.Legacy` | Legacy MVC compatibility example |
| `Example.AzureFunction` | Azure Functions host loading plugins |
| `Example.Akka` | Akka.NET actor system with plugin actors |
| `Example.Avalonia` | Avalonia cross-platform desktop UI with plugins |

### Sample Plugins

| Plugin | Description |
|---|---|
| `Plugin.FileSystem` | File system-based data storage |
| `Plugin.Sql` | SQL database integration |
| `Plugin.SqlClient` | SQL Server client plugin |
| `Plugin.AzureTableStorage` | Azure Table Storage plugin |
| `Plugin.FromHttpBody` | HTTP body parsing plugin |
| `MvcPlugin.DataStorage` | MVC data storage plugin |
| `MvcPlugin.Twitter` | Twitter integration plugin |

### Documentation

- **Main README:** [`README.md`](../README.md) – Project overview, badges, and links
- **Getting Started Guide:** [`docs/README.md`](README.md) – "Weather Project" tutorial
- **Documentation Site:** [https://merken.github.io/Prise](https://merken.github.io/Prise) (GitHub Pages)
