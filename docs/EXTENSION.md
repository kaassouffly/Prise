# Prise – Extension Plan

This document outlines opportunities to extend the Prise plugin framework with new capabilities, platform support, ecosystem integrations, and API enhancements.

---

## Table of Contents

- [1. Executive Summary](#1-executive-summary)
- [2. New Platform Support](#2-new-platform-support)
- [3. Plugin System Enhancements](#3-plugin-system-enhancements)
- [4. API Extensions](#4-api-extensions)
- [5. Ecosystem Integrations](#5-ecosystem-integrations)
- [6. Developer Experience](#6-developer-experience)
- [7. Observability & Diagnostics](#7-observability--diagnostics)
- [8. Security Enhancements](#8-security-enhancements)
- [9. New Sample Applications](#9-new-sample-applications)
- [10. Roadmap](#10-roadmap)

---

## 1. Executive Summary

Prise has a solid foundation for .NET plugin loading with proxy-based type decoupling and assembly isolation. This extension plan identifies opportunities to grow the framework's capabilities while maintaining its core simplicity. Extensions are designed to be opt-in, preserving the lightweight nature of the core framework.

### Extension Philosophy

- **Core stays lean** — New features ship as separate NuGet packages (`Prise.Extensions.*`)
- **Opt-in complexity** — Advanced features never required for basic plugin loading
- **Backwards compatible** — Extensions don't break existing plugin contracts
- **Community-driven** — Each extension has a clear use case and sample application

---

## 2. New Platform Support

### 2.1 .NET MAUI Integration (`Prise.Maui`)

**Priority:** High  
**Audience:** Cross-platform mobile/desktop app developers  
**Status:** Not started

#### Motivation

.NET MAUI has replaced Xamarin and the existing `Example.Avalonia` sample demonstrates demand for UI plugin architectures. A dedicated MAUI integration would enable plugin-based features in mobile and desktop applications.

#### Proposed Package: `Prise.Maui`

```csharp
// Registration in MauiProgram.cs
public static MauiApp CreateMauiApp()
{
    var builder = MauiApp.CreateBuilder();
    builder.UseMauiApp<App>();
    
    builder.Services.AddPrise<IFeaturePlugin>(options => options
        .FromAssembly("FeaturePlugin.dll")
        .DiscoverAt(FileSystem.AppDataDirectory + "/plugins")
        .WithMauiDefaults()  // Platform-aware plugin resolution
    );
    
    return builder.Build();
}
```

#### Key Considerations

- Platform-specific assembly loading (iOS restrictions on dynamic loading)
- Plugin packaging for mobile (embedded resources vs. download)
- Hot-reload support for development workflows
- UI thread marshalling for plugin callbacks

### 2.2 Blazor Integration (`Prise.Blazor`)

**Priority:** High  
**Audience:** Web application developers  
**Status:** Not started

#### Server-Side Blazor

Full plugin loading support using standard `AssemblyLoadContext`:

```csharp
// In Program.cs
builder.Services.AddPrise<IWidgetPlugin>(options => options
    .FromAssembly("WidgetPlugin.dll")
    .DiscoverAt("./plugins/widgets")
    .WithBlazorDefaults()
);

// In a Razor component
@inject IPluginLoader PluginLoader

@code {
    private IWidgetPlugin? widget;
    
    protected override async Task OnInitializedAsync()
    {
        var scan = await PluginLoader.FindPlugin<IWidgetPlugin>("./plugins/widgets");
        widget = await PluginLoader.LoadPlugin<IWidgetPlugin>(scan);
    }
}
```

#### WebAssembly Blazor

Limited support due to WASM restrictions — plugins as pre-compiled Razor Class Libraries:

```csharp
builder.Services.AddPrise<IWidgetPlugin>(options => options
    .FromAssembly("WidgetPlugin.dll")
    .WithWasmMode()  // Pre-loaded assemblies, no dynamic loading
);
```

### 2.3 Minimal API Integration (`Prise.MinimalApi`)

**Priority:** Medium  
**Audience:** Modern ASP.NET Core developers  
**Status:** Not started

#### Motivation

The existing `Prise.Mvc` targets the traditional MVC pipeline. A Minimal API integration would support the modern ASP.NET Core programming model.

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddPrise<IEndpointPlugin>(options => options
    .FromAssembly("ApiPlugin.dll")
    .DiscoverAt("./plugins")
);

var app = builder.Build();

// Auto-register plugin endpoints
app.MapPluginEndpoints<IEndpointPlugin>();

app.Run();
```

#### Plugin Contract

```csharp
public interface IEndpointPlugin
{
    string RoutePrefix { get; }
    void MapEndpoints(IEndpointRouteBuilder routes);
}
```

### 2.4 Worker Service Integration (`Prise.Workers`)

**Priority:** Medium  
**Audience:** Background service developers  
**Status:** Not started

```csharp
// Register plugin-based background services
builder.Services.AddPriseWorker<IBackgroundTaskPlugin>(options => options
    .FromAssembly("TaskPlugin.dll")
    .DiscoverAt("./plugins/tasks")
    .WithPollingInterval(TimeSpan.FromMinutes(5))  // Check for new plugins
);
```

---

## 3. Plugin System Enhancements

### 3.1 Plugin Versioning and Compatibility Checking

**Priority:** High  
**Impact:** Safer plugin upgrades in production

#### Contract Version Metadata

```csharp
// In plugin contract
[PluginContract(Version = "2.0", MinimumHostVersion = "3.0")]
public interface ICalculatorPlugin
{
    Task<decimal> Calculate(string expression);
}

// In plugin implementation
[Plugin(Description = "Advanced Calculator", Version = "2.1")]
public class AdvancedCalculator : ICalculatorPlugin
{
    // ...
}
```

#### Version Compatibility Matrix

```csharp
var scanResult = await loader.FindPlugin<ICalculatorPlugin>("./plugins");

// Check compatibility before loading
if (scanResult.IsCompatibleWith(hostVersion: "3.0"))
{
    var plugin = await loader.LoadPlugin<ICalculatorPlugin>(scanResult);
}
else
{
    logger.LogWarning("Plugin {Name} v{Version} is not compatible with host v3.0",
        scanResult.PluginName, scanResult.PluginVersion);
}
```

### 3.2 Hot Reload / Plugin Watch

**Priority:** High  
**Impact:** Live plugin updates without application restart

```csharp
services.AddPrise<IMyPlugin>(options => options
    .FromAssembly("MyPlugin.dll")
    .DiscoverAt("./plugins")
    .WithHotReload(hotReload => hotReload
        .WatchForChanges()
        .OnPluginUpdated(async (oldPlugin, newPlugin) =>
        {
            logger.LogInformation("Plugin updated: {Name}", newPlugin.GetType().Name);
        })
        .GracefulTransition(TimeSpan.FromSeconds(30))  // Drain existing requests
    )
);
```

#### Implementation Approach

1. Use `FileSystemWatcher` to monitor plugin directories
2. On file change, load new assembly into fresh `AssemblyLoadContext`
3. Swap proxy targets atomically
4. Unload old `AssemblyLoadContext` after graceful drain period
5. Emit lifecycle events for monitoring

### 3.3 Plugin Dependency Injection Scopes

**Priority:** Medium  
**Impact:** Better service lifetime management for plugins

#### Current State

Plugins receive services via `[PluginService]` attribute injection. Service lifetimes are not well-defined.

#### Proposed Enhancement

```csharp
services.AddPrise<IMyPlugin>(options => options
    .FromAssembly("MyPlugin.dll")
    .WithPluginServices(pluginServices =>
    {
        // Scoped to plugin lifetime
        pluginServices.AddScoped<IPluginLogger, PluginLogger>();
        
        // Shared across all plugins (singleton)
        pluginServices.AddSingleton<ISharedCache>(hostCache);
        
        // Transient per invocation
        pluginServices.AddTransient<IRequestContext, PluginRequestContext>();
    })
);
```

### 3.4 Remote Plugin Loading

**Priority:** Medium  
**Impact:** Enables distributed plugin architectures

#### Plugin Sources

```csharp
services.AddPrise<IMyPlugin>(options => options
    .FromAssembly("MyPlugin.dll")
    // Load from various sources
    .DiscoverFromFileSystem("./plugins")
    .DiscoverFromNuGet("https://nuget.org/v3/index.json", "MyPlugin", "1.0.0")
    .DiscoverFromHttp("https://plugins.example.com/MyPlugin.dll")
    .DiscoverFromAzureBlobStorage(connectionString, "plugins-container")
);
```

#### Security Considerations

- Assembly signature verification before loading
- Sandboxed execution context for untrusted plugins
- Network timeout and retry policies
- Checksum verification for downloaded assemblies

### 3.5 Plugin Communication (Inter-Plugin Messaging)

**Priority:** Low  
**Impact:** Enables complex plugin ecosystems

```csharp
// Plugin A publishes events
[Plugin]
public class DataPlugin : IDataPlugin
{
    [PluginService] private IPluginMessageBus _messageBus;
    
    public async Task ProcessData(Data data)
    {
        await _messageBus.PublishAsync(new DataProcessedEvent(data));
    }
}

// Plugin B subscribes to events
[Plugin]
public class NotificationPlugin : INotificationPlugin, IPluginMessageHandler<DataProcessedEvent>
{
    public async Task HandleAsync(DataProcessedEvent message)
    {
        await SendNotification(message.Data);
    }
}
```

---

## 4. API Extensions

### 4.1 Fluent Discovery API

**Priority:** High  
**Impact:** Improved developer experience

```csharp
// Current
var scanResult = await loader.FindPlugin<IMyPlugin>("./plugins");
var plugin = await loader.LoadPlugin<IMyPlugin>(scanResult);

// Proposed fluent API
var plugin = await loader
    .Discover<IMyPlugin>()
    .In("./plugins")
    .Where(p => p.Version >= "2.0")
    .OrderBy(p => p.Version, descending: true)
    .LoadFirst();

// Load all matching plugins
var plugins = await loader
    .Discover<IMyPlugin>()
    .In("./plugins")
    .LoadAll();

// With async enumeration
await foreach (var plugin in loader.Discover<IMyPlugin>().In("./plugins").StreamAll())
{
    await plugin.Execute();
}
```

### 4.2 Plugin Metadata API

**Priority:** Medium  
**Impact:** Plugin management UIs, admin dashboards

```csharp
public interface IPluginMetadata
{
    string Name { get; }
    string Version { get; }
    string Description { get; }
    string Author { get; }
    IReadOnlyList<string> Tags { get; }
    IReadOnlyList<PluginDependencyInfo> Dependencies { get; }
    DateTimeOffset? BuildDate { get; }
    string? SourceHash { get; }
}

// Usage
var metadata = await loader.GetPluginMetadata<IMyPlugin>(scanResult);
Console.WriteLine($"Plugin: {metadata.Name} v{metadata.Version} by {metadata.Author}");
```

### 4.3 Plugin Health Checks

**Priority:** Medium  
**Impact:** Production monitoring, reliability

```csharp
// ASP.NET Core health check integration
builder.Services.AddHealthChecks()
    .AddPrisePluginHealthCheck<IMyPlugin>("my-plugin", options =>
    {
        options.Timeout = TimeSpan.FromSeconds(5);
        options.FailureStatus = HealthStatus.Degraded;
    });

// Plugin implements health check
[Plugin]
public class MyPlugin : IMyPlugin, IPluginHealthCheck
{
    public async Task<PluginHealthResult> CheckHealthAsync(CancellationToken ct)
    {
        // Verify plugin dependencies are available
        var dbConnection = await TryConnectToDatabase();
        return dbConnection.IsSuccessful
            ? PluginHealthResult.Healthy()
            : PluginHealthResult.Unhealthy("Database unavailable");
    }
}
```

### 4.4 Cancellation Token Support

**Priority:** High  
**Impact:** Proper async cancellation throughout the pipeline

#### Current State

Most async methods in `IPluginLoader` do not accept `CancellationToken` parameters.

#### Proposed Extension

```csharp
public interface IPluginLoader : IDisposable, IAsyncDisposable
{
    // Existing methods with CancellationToken overloads
    Task<AssemblyScanResult> FindPlugin<T>(
        string pathToPlugins,
        CancellationToken cancellationToken = default);
    
    Task<T> LoadPlugin<T>(
        AssemblyScanResult scanResult,
        string? hostFramework = null,
        Action<PluginLoadContext>? configure = null,
        CancellationToken cancellationToken = default);
    
    Task UnloadAsync(CancellationToken cancellationToken = default);
}
```

---

## 5. Ecosystem Integrations

### 5.1 OpenTelemetry Integration (`Prise.OpenTelemetry`)

**Priority:** High  
**Impact:** Production-grade observability

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddPriseInstrumentation()  // Auto-instrument plugin operations
    )
    .WithMetrics(metrics => metrics
        .AddPriseInstrumentation()  // Plugin load times, invocation counts
    );
```

#### Traces

| Span Name | Attributes |
|---|---|
| `prise.plugin.scan` | `plugin.type`, `plugin.path`, `plugin.count` |
| `prise.plugin.load` | `plugin.name`, `plugin.version`, `assembly.name` |
| `prise.plugin.activate` | `plugin.name`, `activation.duration_ms` |
| `prise.plugin.invoke` | `plugin.name`, `method.name`, `method.duration_ms` |
| `prise.plugin.unload` | `plugin.name`, `unload.duration_ms` |

#### Metrics

| Metric | Type | Description |
|---|---|---|
| `prise.plugins.loaded` | UpDownCounter | Currently loaded plugins |
| `prise.plugin.load.duration` | Histogram | Plugin load duration |
| `prise.plugin.invoke.duration` | Histogram | Plugin method invocation duration |
| `prise.plugin.errors` | Counter | Plugin error count |

### 5.2 Source Generator for Plugin Contracts (`Prise.Generators`)

**Priority:** Medium  
**Impact:** Compile-time validation, reduced boilerplate

```csharp
// Source generator creates plugin metadata at compile time
[GeneratePluginContract]
public interface ICalculatorPlugin
{
    Task<decimal> Calculate(string expression);
}

// Generated code (auto):
// - Parameter/Result DTOs for proxy serialization
// - Contract version hash
// - Validation attributes
// - IntelliSense documentation
```

### 5.3 gRPC Plugin Communication (`Prise.Grpc`)

**Priority:** Low  
**Impact:** High-performance cross-process plugin communication

For scenarios where plugins run in separate processes:

```csharp
services.AddPrise<IMyPlugin>(options => options
    .FromAssembly("MyPlugin.dll")
    .UseGrpcTransport(grpc => grpc
        .Endpoint("https://localhost:5001")
        .WithChannelOptions(new GrpcChannelOptions
        {
            MaxRetryAttempts = 3
        })
    )
);
```

### 5.4 Configuration Provider Integration

**Priority:** Medium  
**Impact:** Plugin-specific configuration via standard .NET config

```csharp
// appsettings.json
{
  "Prise": {
    "Plugins": {
      "CalculatorPlugin": {
        "Assembly": "Calculator.dll",
        "Path": "./plugins/calculator",
        "Enabled": true,
        "Settings": {
          "Precision": 10,
          "MaxExpression": "1000"
        }
      }
    }
  }
}

// Registration
builder.Services.AddPrise<ICalculatorPlugin>(
    builder.Configuration.GetSection("Prise:Plugins:CalculatorPlugin"));
```

---

## 6. Developer Experience

### 6.1 dotnet CLI Tool (`dotnet-prise`)

**Priority:** Medium  
**Impact:** Streamlined plugin development workflow

```bash
# Create a new plugin project
dotnet prise new plugin --name MyPlugin --contract IMyPlugin

# Package a plugin for deployment
dotnet prise pack --output ./dist

# Validate a plugin against a contract
dotnet prise validate --plugin ./MyPlugin.dll --contract ./Contract.dll

# List plugins in a directory
dotnet prise list ./plugins

# Scan for compatibility issues
dotnet prise check --plugin ./MyPlugin.dll --host-framework net8.0
```

### 6.2 Visual Studio / Rider Extension

**Priority:** Low  
**Impact:** Enhanced IDE integration

- Plugin project templates
- Contract/Plugin relationship visualization
- One-click plugin deployment to host
- Debugging support across plugin boundaries

### 6.3 Plugin Project Templates

**Priority:** Medium  
**Impact:** Faster onboarding for new plugin developers

```bash
# Install templates
dotnet new install Prise.Templates

# Available templates:
# prise-plugin         Basic plugin project
# prise-plugin-mvc     MVC controller plugin
# prise-plugin-api     Minimal API endpoint plugin
# prise-host-web       Web host with plugin support
# prise-host-console   Console host with plugin support
```

Template structure:

```
templates/
├── prise-plugin/
│   ├── .template.config/template.json
│   ├── PluginName.csproj
│   ├── MyPlugin.cs
│   └── PluginBootstrapper.cs
├── prise-host-web/
│   ├── .template.config/template.json
│   ├── HostName.csproj
│   ├── Program.cs
│   └── appsettings.json
└── ...
```

---

## 7. Observability & Diagnostics

### 7.1 Diagnostic Events via EventSource

**Priority:** Medium  
**Impact:** Low-overhead production diagnostics

```csharp
[EventSource(Name = "Prise.PluginFramework")]
internal sealed class PriseEventSource : EventSource
{
    public static readonly PriseEventSource Instance = new();

    [Event(1, Level = EventLevel.Informational, Message = "Plugin scan started: {0}")]
    public void PluginScanStarted(string path) => WriteEvent(1, path);

    [Event(2, Level = EventLevel.Informational, Message = "Plugin loaded: {0} v{1}")]
    public void PluginLoaded(string name, string version) => WriteEvent(2, name, version);

    [Event(3, Level = EventLevel.Error, Message = "Plugin load failed: {0} - {1}")]
    public void PluginLoadFailed(string name, string error) => WriteEvent(3, name, error);

    [Event(4, Level = EventLevel.Informational, Message = "Plugin unloaded: {0}")]
    public void PluginUnloaded(string name) => WriteEvent(4, name);
}
```

Consumption with `dotnet-counters`:

```bash
dotnet counters monitor --process-id <PID> Prise.PluginFramework
```

### 7.2 Plugin Dependency Graph Visualization

**Priority:** Low  
**Impact:** Debugging complex plugin dependency issues

```csharp
// Generate dependency graph
var graph = await loader.GetDependencyGraph<IMyPlugin>(scanResult);

// Export as DOT format for Graphviz
var dot = graph.ToDotFormat();
File.WriteAllText("plugin-deps.dot", dot);

// Or as JSON for web visualization
var json = graph.ToJson();
```

### 7.3 Plugin Load Diagnostics

**Priority:** Medium  
**Impact:** Faster troubleshooting of plugin load failures

```csharp
// Detailed diagnostic information
var diagnostics = await loader.DiagnosePlugin<IMyPlugin>("./plugins/MyPlugin.dll");

Console.WriteLine($"Assembly: {diagnostics.AssemblyName}");
Console.WriteLine($"Contract Match: {diagnostics.ContractMatch}");
Console.WriteLine($"Missing Dependencies: {string.Join(", ", diagnostics.MissingDependencies)}");
Console.WriteLine($"Version Conflicts: {string.Join(", ", diagnostics.VersionConflicts)}");
Console.WriteLine($"Native Libraries: {string.Join(", ", diagnostics.NativeLibraries)}");
Console.WriteLine($"Load Strategy: {diagnostics.RecommendedLoadStrategy}");
```

---

## 8. Security Enhancements

### 8.1 Plugin Signature Verification

**Priority:** High  
**Impact:** Prevents loading of tampered or unauthorized plugins

```csharp
services.AddPrise<IMyPlugin>(options => options
    .FromAssembly("MyPlugin.dll")
    .DiscoverAt("./plugins")
    .WithSignatureVerification(signing => signing
        .RequireStrongName()
        .AllowPublicKeys("key1.snk", "key2.snk")
        .RejectUnsigned()
    )
);
```

### 8.2 Plugin Sandboxing

**Priority:** Medium  
**Impact:** Limits plugin capabilities for untrusted sources

```csharp
services.AddPrise<IMyPlugin>(options => options
    .FromAssembly("MyPlugin.dll")
    .WithSandbox(sandbox => sandbox
        .DenyFileSystemAccess()
        .DenyNetworkAccess()
        .AllowedAssemblies("System.Text.Json", "Microsoft.Extensions.Logging")
        .MaxMemory(megabytes: 100)
        .MaxCpuTime(TimeSpan.FromSeconds(30))
    )
);
```

### 8.3 Plugin Permission Model

**Priority:** Low  
**Impact:** Granular control over plugin capabilities

```csharp
// Plugin declares required permissions
[Plugin]
[RequiresPermission(PluginPermission.FileSystem, Reason = "Reads configuration files")]
[RequiresPermission(PluginPermission.Network, Reason = "Calls external API")]
public class MyPlugin : IMyPlugin { }

// Host grants permissions
services.AddPrise<IMyPlugin>(options => options
    .WithPermissions(permissions => permissions
        .Grant(PluginPermission.FileSystem)
        .Deny(PluginPermission.Network)
    )
);
```

---

## 9. New Sample Applications

### 9.1 Proposed Samples

| Sample | Description | Demonstrates |
|---|---|---|
| `Example.MinimalApi` | Minimal API with plugin endpoints | `Prise.MinimalApi` |
| `Example.Blazor.Server` | Blazor Server with plugin widgets | `Prise.Blazor` |
| `Example.Blazor.Wasm` | Blazor WASM with plugin modules | `Prise.Blazor` WASM mode |
| `Example.MAUI` | .NET MAUI app with feature plugins | `Prise.Maui` |
| `Example.Worker` | Background service with task plugins | `Prise.Workers` |
| `Example.Microservice` | Microservice with plugin middleware | Hot reload, health checks |
| `Example.MultiTenant` | Multi-tenant app with tenant plugins | Plugin scoping, isolation |
| `Example.Marketplace` | Plugin marketplace UI | Metadata API, versioning |

### 9.2 Updated Existing Samples

| Sample | Update |
|---|---|
| `Example.Console` | Add hot reload, cancellation token |
| `Example.Web` | Add health checks, OpenTelemetry |
| `Example.Mvc.Controllers` | Migrate to Minimal API alternative |
| `Example.AzureFunction` | Update to isolated worker model |

---

## 10. Roadmap

### Version 3.0 (Next Major)

**Theme:** Modern .NET Alignment

- [ ] Drop EOL framework targets (see [MODERNIZATION.md](MODERNIZATION.md))
- [ ] Add net8.0 and net9.0 support
- [ ] Enable nullable reference types everywhere
- [ ] Add `CancellationToken` to all async APIs
- [ ] Add `IAsyncDisposable` support
- [ ] Plugin versioning and compatibility metadata
- [ ] Remove deprecated APIs (`BridgeType`, `PluginFactory`)

### Version 3.1

**Theme:** Developer Experience

- [ ] Fluent discovery API
- [ ] Plugin metadata API
- [ ] dotnet CLI tool (`dotnet-prise`)
- [ ] Project templates
- [ ] Comprehensive XML documentation
- [ ] CONTRIBUTING.md and updated docs

### Version 3.2

**Theme:** Production Readiness

- [ ] Hot reload / plugin watch
- [ ] Plugin health checks
- [ ] OpenTelemetry integration
- [ ] EventSource diagnostics
- [ ] Plugin load diagnostics
- [ ] Structured logging throughout

### Version 4.0

**Theme:** Platform Expansion

- [ ] Blazor integration (`Prise.Blazor`)
- [ ] Minimal API integration (`Prise.MinimalApi`)
- [ ] Worker Service integration (`Prise.Workers`)
- [ ] .NET MAUI integration (`Prise.Maui`)
- [ ] Plugin signature verification
- [ ] Remote plugin loading (HTTP, NuGet, Azure Blob)

### Version 5.0

**Theme:** Advanced Scenarios

- [ ] Source generators for plugin contracts
- [ ] Inter-plugin messaging
- [ ] Plugin sandboxing
- [ ] gRPC transport for cross-process plugins
- [ ] Plugin permission model
- [ ] Plugin dependency graph visualization
