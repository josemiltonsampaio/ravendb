# Raven.Embedded - Embedded Database Server

## ?? Visão Geral

O **Raven.Embedded** é uma biblioteca que permite embedar o servidor RavenDB completo dentro de aplicações .NET, eliminando a necessidade de instalação e gerenciamento separado do servidor. É ideal para aplicações desktop, testes de integração e cenários onde simplicidade de deployment é crítica.

### Propósito

- **Embedar RavenDB** em aplicações .NET (desktop, console, ASP.NET)
- **Simplificar deployment** (zero-configuration, self-contained)
- **Facilitar desenvolvimento** e testes
- **Gerenciar lifecycle** do servidor automaticamente
- **Process isolation** (servidor roda em processo separado)

### Características Principais

```
Multi-Target: .NET 6.0, 7.0, 8.0, .NET Standard 2.0, .NET Framework 4.6.2
Arquivos: ~12 arquivos .cs
LOC: ~1.500 linhas de código
Servidor completo embedded (não é in-process, mas out-of-process)
```

---

## ??? Arquitetura Interna

### Design Pattern: Singleton + Process Management

```
??????????????????????????????????????????????????
?         EmbeddedServer (Singleton)             ?
?  - static Instance                             ?
?  - Process Management                          ?
?  - DocumentStore Factory                       ?
??????????????????????????????????????????????????
               ?
    ?????????????????????????
    ?                       ?
????????????????   ????????????????????????
?ServerOptions ?   ?  RavenServerRunner   ?
?- Config      ?   ?  - Process Startup   ?
?- Security    ?   ?  - Args Builder      ?
?- Licensing   ?   ?  - Env Vars          ?
????????????????   ????????????????????????
                          ?
                   ?????????????????
                   ?  Raven.Server ?
                   ?   (Process)   ?
                   ?????????????????
```

### Componentes Principais

1. **EmbeddedServer** - Singleton que gerencia lifecycle do servidor
2. **RavenServerRunner** - Responsável por lançar o processo do servidor
3. **ServerOptions** - Configuração do servidor
4. **DatabaseOptions** - Configuração de databases individuais
5. **RuntimeFrameworkVersionMatcher** - Seleciona versão correta do .NET runtime

---

## ?? Técnicas de Implementação

### 1. Singleton Pattern com Lazy Initialization

```csharp
// src/Raven.Embedded/EmbeddedServer.cs
public sealed class EmbeddedServer : IDisposable
{
    // Singleton instance - thread-safe via CLR
    public static EmbeddedServer Instance = new EmbeddedServer();

    // Private constructor - prevents direct instantiation
    internal EmbeddedServer() { }
    
    // Lazy server startup
    private Lazy<Task<(Uri ServerUrl, Process ServerProcess)>>? _serverTask;
    
    // Lazy document stores (one per database)
    private readonly ConcurrentDictionary<string, Lazy<Task<IDocumentStore>>> 
        _documentStores = new ConcurrentDictionary<string, Lazy<Task<IDocumentStore>>>();
}
```

**?? Por que Singleton?**
- **Único servidor** por aplicação (resource sharing)
- **Gerenciamento centralizado** de lifecycle
- **Evita múltiplos processos** do servidor
- **Thread-safe** via CLR initialization

**?? Lazy Initialization Benefits:**
- **Start on-demand** (não ao carregar assembly)
- **Async startup** (não bloqueia thread principal)
- **Exception handling** (captura erros de startup)

---

### 2. Out-of-Process Architecture

```csharp
// O servidor NÃO roda in-process, mas como processo separado!
private async Task<(Uri ServerUrl, Process ServerProcess)> RunServer()
{
    if (_serverOptions == null)
        throw new ArgumentNullException(nameof(_serverOptions));

    // Lança Raven.Server.exe (ou dotnet Raven.Server.dll)
    var process = await RavenServerRunner.RunAsync(_serverOptions).ConfigureAwait(false);
    
    // Registra cleanup no AppDomain/AssemblyLoadContext unload
#if NET462
    AppDomain.CurrentDomain.DomainUnload += (s, args) =>
    {
        ShutdownServerProcess(process);
    };
#else
    AssemblyLoadContext.Default.Unloading += c =>
    {
        ShutdownServerProcess(process);
    };
#endif

    // Aguarda servidor estar pronto (lê stdout)
    var stdoutTcs = new TaskCompletionSource<(string Url, string Stdout)>(...);
    
    process.OutputDataReceived += (_, receivedEventArgs) =>
    {
        const string prefix = "Server available on: ";
        if (receivedEventArgs.Data.StartsWith(prefix))
        {
            var url = receivedEventArgs.Data.Substring(prefix.Length);
            stdoutTcs.TrySetResult((url, ...));
        }
    };
    
    process.BeginOutputReadLine();
    
    // Wait for server to start (with timeout)
    var timeoutTask = Task.Delay(_serverOptions.MaxServerStartupTimeDuration);
    var firstCompleted = await Task.WhenAny(stdoutTcs.Task, timeoutTask);
    
    if (firstCompleted == stdoutTcs.Task)
    {
        (string url, _) = stdoutTcs.Task.Result;
        return (new Uri(url), process);
    }
    
    // Timeout!
    ShutdownServerProcess(process);
    throw new TimeoutException($"Server failed to start in {MaxServerStartupTimeDuration}");
}
```

**?? Out-of-Process Advantages:**

| Aspect | In-Process | Out-of-Process (Raven.Embedded) |
|--------|------------|----------------------------------|
| **Isolation** | ? Shared AppDomain | ? Separate process |
| **Crash Impact** | ? Kills host app | ? App can recover |
| **Memory** | ? Same heap | ? Separate memory space |
| **Debugging** | ? Same debugger | ? Need attach to process |
| **Performance** | ? No IPC overhead | ? HTTP/TCP communication |
| **Updates** | ? Requires app restart | ? Can restart server only |

**?? Por que Out-of-Process?**
1. **Isolation:** Crash do servidor não mata a aplicação
2. **Resources:** Servidor pode ter seu próprio thread pool, GC, etc.
3. **Simplicity:** Usa mesmo servidor que produção
4. **Flexibility:** Pode reiniciar servidor sem reiniciar app

---

### 3. Process Lifecycle Management

#### Graceful Shutdown

```csharp
private void ShutdownServerProcess(Process process)
{
    if (process == null || process.HasExited)
        return;

    lock (process)
    {
        if (process.HasExited)
            return;

        try
        {
            // Attempt graceful shutdown via stdin
            using (var inputStream = process.StandardInput)
            {
                inputStream.WriteLine("shutdown no-confirmation");
            }

            // Wait for graceful shutdown
            if (process.WaitForExit((int)_serverOptions.GracefulShutdownTimeout.TotalMilliseconds))
                return; // Success!
        }
        catch (Exception e)
        {
            if (_logger.IsWarnEnabled)
                _logger.Warn($"Failed to shutdown server gracefully", e);
        }

        // Graceful shutdown failed - force kill
        KillServerProcess(process);
    }
}

private void KillServerProcess(Process process)
{
    if (process == null || process.HasExited)
        return;

    lock (process) // Double-check pattern
    {
        if (process.HasExited)
            return;

        try
        {
            process.Kill(); // Force terminate
        }
        catch (Exception e)
        {
            _logger.Warn($"Failed to kill process {process.Id}", e);
        }
    }
}
```

**?? Two-Phase Shutdown:**
1. **Phase 1 (Graceful):** Send `shutdown` command via stdin
2. **Phase 2 (Force):** Kill process if graceful fails

**?? Locking Strategy:**
- `lock (process)` prevents **race conditions** entre threads
- **Double-check** `HasExited` dentro do lock
- Evita **multiple kill attempts**

---

### 4. Server Process Arguments Builder

```csharp
// src/Raven.Embedded/RavenServerRunner.cs
public static async Task<Process> RunAsync(ServerOptions options)
{
    // Determine executable (exe vs dll)
    (string exec, string fstArg) = GetExecAndFirstArgument(options);

    var commandLineArgs = new List<string>(options.CommandLineArgs);

    // Parent process monitoring (auto-shutdown se parent morrer)
    using (var currentProcess = Process.GetCurrentProcess())
    {
        commandLineArgs.Add($"--Embedded.ParentProcessId={currentProcess.Id}");
    }

    // Licensing
    if (options.Licensing != null)
    {
        if (!string.IsNullOrWhiteSpace(options.Licensing.License))
            commandLineArgs.Add($"--License={EscapeSingleArg(options.Licensing.License)}");
        
        commandLineArgs.Add($"--License.Eula.Accepted={options.Licensing.EulaAccepted}");
        // ... more licensing args
    }

    // Core settings
    commandLineArgs.Add("--Setup.Mode=None"); // Disable setup wizard
    commandLineArgs.Add($"--DataDir={EscapeSingleArg(options.DataDirectory)}");
    commandLineArgs.Add($"--Logs.Path={EscapeSingleArg(options.LogsPath)}");

    // Security
    if (options.Security != null)
    {
        options.ServerUrl = "https://127.0.0.1:0"; // Random HTTPS port
        
        if (options.Security.CertificatePath != null)
        {
            commandLineArgs.Add($"--Security.Certificate.Path={EscapeSingleArg(options.Security.CertificatePath)}");
            if (options.Security.CertificatePassword != null)
                commandLineArgs.Add($"--Security.Certificate.Password={...}");
        }
        
        // Set client certificate as admin
        commandLineArgs.Add($"--Security.WellKnownCertificates.Admin={thumbprint}");
    }
    else
    {
        options.ServerUrl = "http://127.0.0.1:0"; // Random HTTP port
    }

    commandLineArgs.Add($"--ServerUrl={options.ServerUrl}");

    // Framework version matching (se necessário)
    if (fstArg != null) // Using dotnet.exe to launch dll
    {
        commandLineArgs.Insert(0, EscapeSingleArg(fstArg));
        
        if (!string.IsNullOrWhiteSpace(options.FrameworkVersion))
        {
            var frameworkVersion = await RuntimeFrameworkVersionMatcher.MatchAsync(options);
            commandLineArgs.Insert(0, $"--fx-version {frameworkVersion}");
        }
    }

    // Build final command
    var processStartInfo = new ProcessStartInfo
    {
        FileName = exec,
        Arguments = string.Join(" ", commandLineArgs),
        CreateNoWindow = true,
        RedirectStandardOutput = true,
        RedirectStandardError = true,
        RedirectStandardInput = true,
        UseShellExecute = false
    };

    // Clean environment variables (remove IIS/ASP.NET vars)
    RemoveEnvironmentVariables(processStartInfo);

    var process = Process.Start(processStartInfo);
    process.EnableRaisingEvents = true;
    
    return process;
}
```

**?? Command Line Building:**
- **Escape arguments** properly (evita command injection)
- **Parent process monitoring** (auto-shutdown orphan server)
- **Dynamic port allocation** (0 = OS escolhe porta livre)
- **Framework version matching** (compatibilidade .NET)
- **Environment cleanup** (remove vars do IIS/ASP.NET)

---

### 5. Runtime Framework Version Matching

```csharp
// src/Raven.Embedded/RuntimeFrameworkVersionMatcher.cs
internal static async Task<string> MatchAsync(ServerOptions options)
{
    if (NeedsMatch(options) == false)
        return options?.FrameworkVersion;

    // Parse requested version (pode ter wildcards: "8.0.x" ou "8.0.22+")
    var runtime = new RuntimeFrameworkVersion(options.FrameworkVersion);
    
    // Get installed runtimes via 'dotnet --info'
    var runtimes = await GetFrameworkVersionsAsync(options).ConfigureAwait(false);

    // Match based on wildcards
    return Match(runtime, runtimes);
}

internal static string Match(RuntimeFrameworkVersion runtime, List<RuntimeFrameworkVersion> runtimes)
{
    // Sort descending (prefer newer versions)
    var sortedRuntimes = runtimes
        .OrderByDescending(x => x.Major)
        .ThenByDescending(x => x.Minor)
        .ThenByDescending(x => x.Patch)
        .ToList();

    foreach (var version in sortedRuntimes)
    {
        if (runtime.Match(version)) // Supports wildcards e GreaterOrEqual
            return version.ToString();
    }

    // No match found!
    throw new InvalidOperationException($"Could not find matching runtime for '{runtime}'");
}
```

**?? Version Matching Examples:**

```csharp
// Exact version
FrameworkVersion = "8.0.22"  ? Matches 8.0.22 only

// Wildcard patch
FrameworkVersion = "8.0.x"   ? Matches any 8.0.* (highest installed)

// Greater or equal
FrameworkVersion = "8.0.22+" ? Matches 8.0.22, 8.0.23, 8.0.100, etc (highest)

// Combined
FrameworkVersion = "8.x.x+"  ? Matches any 8.* runtime (highest)
```

**?? Por que Version Matching?**
- **Compatibility:** Garantir que servidor roda em runtime compatível
- **Flexibility:** Suporta wildcards para "latest patch"
- **Forward-compatible:** `+` permite versões futuras
- **Error early:** Falha rápida se runtime não disponível

---

### 6. Document Store Factory Pattern

```csharp
public async Task<IDocumentStore> GetDocumentStoreAsync(DatabaseOptions options, CancellationToken token = default)
{
    var databaseName = options.DatabaseRecord.DatabaseName;
    
    if (string.IsNullOrWhiteSpace(databaseName))
        throw new ArgumentNullException(nameof(databaseName));

    if (_serverOptions == null)
        throw new InvalidOperationException("Server not started");

    // Lazy factory para cada database
    var lazy = new Lazy<Task<IDocumentStore>>(async () =>
    {
        var serverUrl = await GetServerUriAsync(token).ConfigureAwait(false);
        
        var store = new DocumentStore
        {
            Urls = new[] { serverUrl.AbsoluteUri },
            Database = databaseName,
            Certificate = _serverOptions.Security?.ClientCertificate,
            Conventions = options.Conventions
        };

        // CRITICAL: Disable topology cache (server is local, no failover needed)
        store.Conventions.DisableTopologyCache = true;

        // Cleanup on dispose
        store.AfterDispose += (sender, args) => _documentStores.TryRemove(databaseName, out _);

        store.Initialize();
        
        // Create database if not exists
        if (options.SkipCreatingDatabase == false)
            await TryCreateDatabase(options, store, token).ConfigureAwait(false);

        return store;
    });

    // Cache store per database (reuse if already created)
    return await _documentStores.GetOrAdd(databaseName, lazy).Value.WithCancellation(token).ConfigureAwait(false);
}
```

**?? Design Patterns:**

1. **Factory Pattern:** Creates DocumentStore instances
2. **Cache Pattern:** Reuses stores for same database
3. **Lazy Initialization:** Store created on first access
4. **Disposal Hook:** Auto-cleanup on store dispose

**?? Important Settings:**

```csharp
store.Conventions.DisableTopologyCache = true;
```

**Por que?**
- Servidor é **local** (não há cluster)
- Topology **não muda** (processo único)
- **Evita overhead** de topology updates
- **Simplifica** comunicação

---

### 7. Graceful Database Creation

```csharp
private async Task TryCreateDatabase(DatabaseOptions options, IDocumentStore store, CancellationToken token)
{
    try
    {
        await store.Maintenance.Server.SendAsync(
            new CreateDatabaseOperation(options.DatabaseRecord), 
            token
        ).ConfigureAwait(false);
    }
    catch (ConcurrencyException)
    {
        // Expected - database already exists
        // Multiple concurrent GetDocumentStore() calls podem chegar aqui
        if (_logger.IsDebugEnabled)
            _logger.Debug($"{options.DatabaseRecord.DatabaseName} already exists.");
    }
}
```

**?? Idempotent Operation:**
- **ConcurrencyException** é esperada (não é erro)
- **Permite múltiplas chamadas** concorrentes
- **Idempotent:** Múltiplas calls = mesmo resultado

---

## ?? Padrões Arquiteturais

### 1. Singleton Pattern

```csharp
public static EmbeddedServer Instance = new EmbeddedServer();
```

**Justificativa:**
- Apenas um servidor por aplicação
- Shared resource (process, ports)
- Thread-safe initialization (CLR guarantee)

---

### 2. Lazy Initialization Pattern

```csharp
// Server startup
private Lazy<Task<(Uri, Process)>>? _serverTask;

// Document stores
private readonly ConcurrentDictionary<string, Lazy<Task<IDocumentStore>>> _documentStores;
```

**Benefits:**
- **Start on-demand** (não ao carregar assembly)
- **Async support** (Lazy<Task<T>>)
- **Thread-safe** (Lazy handles concurrency)
- **Exception caching** (Lazy caches exceptions)

---

### 3. Factory Pattern

```csharp
public Task<IDocumentStore> GetDocumentStoreAsync(DatabaseOptions options)
{
    var lazy = new Lazy<Task<IDocumentStore>>(async () =>
    {
        // Create and configure store
        var store = new DocumentStore { ... };
        store.Initialize();
        return store;
    });

    return _documentStores.GetOrAdd(databaseName, lazy).Value;
}
```

---

### 4. Process Wrapper Pattern

```csharp
// Wraps Process with lifecycle management
private async Task<(Uri ServerUrl, Process ServerProcess)> RunServer()
{
    var process = await RavenServerRunner.RunAsync(_serverOptions);
    
    // Hook cleanup events
    AppDomain.CurrentDomain.DomainUnload += (s, args) => ShutdownServerProcess(process);
    
    // Monitor stdout for "ready" signal
    process.OutputDataReceived += ...;
    
    return (serverUrl, process);
}
```

---

## ?? Técnicas de Otimização

### 1. Command Line Argument Escaping

```csharp
// Uses CommandLineArgumentEscaper do Sparrow
commandLineArgs.Add($"--DataDir={CommandLineArgumentEscaper.EscapeSingleArg(options.DataDirectory)}");
```

**Evita:**
- **Command injection** vulnerabilities
- **Parsing errors** (espaços, quotes, etc.)
- **Cross-platform issues** (Windows vs Linux)

---

### 2. Environment Variable Cleanup

```csharp
private static void RemoveEnvironmentVariables(ProcessStartInfo processStartInfo)
{
    var variablesToRemove = new List<string>();
    
    foreach (var key in processStartInfo.Environment.Keys)
    {
        if (key.StartsWith("APP_POOL_", StringComparison.OrdinalIgnoreCase) ||
            key.StartsWith("ASPNETCORE_", StringComparison.OrdinalIgnoreCase) ||
            key.StartsWith("IIS_", StringComparison.OrdinalIgnoreCase))
        {
            variablesToRemove.Add(key);
        }
    }

    foreach (var key in variablesToRemove)
        processStartInfo.Environment.Remove(key);
}
```

**?? Por que remover?**
- **IIS vars** confundem o servidor embedded
- **ASP.NET Core vars** podem causar conflitos
- **APP_POOL vars** são irrelevantes para embedded

---

### 3. Async Stdout/Stderr Reading

```csharp
var stderrBuilder = new StringBuilder();
var stdoutBuilder = new StringBuilder();

process.ErrorDataReceived += (_, e) =>
{
    if (e.Data == null) return;
    lock (stderrBuilder)
        stderrBuilder.AppendLine(e.Data);
};

process.OutputDataReceived += (_, e) =>
{
    if (e.Data == null) return;
    lock (stdoutBuilder)
        stdoutBuilder.AppendLine(e.Data);
    
    // Detect "Server available on: ..." line
    const string prefix = "Server available on: ";
    if (e.Data.StartsWith(prefix))
    {
        var url = e.Data.Substring(prefix.Length);
        stdoutTcs.TrySetResult((url, ...));
    }
};

process.BeginErrorReadLine();
process.BeginOutputReadLine();
```

**?? Async Benefits:**
- **Non-blocking** read (não trava thread)
- **Real-time** output processing
- **Detect ready state** imediatamente

**?? Thread Safety:**
- **lock (stderrBuilder)** previne races entre callbacks
- Callbacks executam em **ThreadPool threads**

---

### 4. Timeout Handling com Task.WhenAny

```csharp
var timeoutTask = Task.Delay(_serverOptions.MaxServerStartupTimeDuration);

var firstCompleted = await Task.WhenAny(stdoutTcs.Task, stderrTcs.Task, timeoutTask);

if (firstCompleted == stdoutTcs.Task)
{
    // Success - server started
    return (url, process);
}
else if (firstCompleted == stderrTcs.Task)
{
    // Error output received
    throw new InvalidOperationException(stderrString);
}
else
{
    // Timeout!
    throw new TimeoutException($"Server failed to start in {MaxServerStartupTimeDuration}");
}
```

**?? Pattern: Task.WhenAny para Race Conditions**
- **Primeira task** que completa vence
- **Timeout** é apenas mais uma task na race
- **Clean code:** Sem `CancellationToken` explícito

---

## ??? Tratamento de Erros

### 1. Startup Failure Handling

```csharp
private Exception BuildServerStartupException(string? outputString, string? errorString, bool isTimeout)
{
    var sb = new StringBuilder();
    
    sb.AppendLine(isTimeout
        ? $"Server failed to start in {_serverOptions.MaxServerStartupTimeDuration}"
        : "Unable to start the RavenDB Server");

    if (!string.IsNullOrWhiteSpace(errorString))
    {
        sb.AppendLine("Error:");
        sb.AppendLine(errorString);
    }

    if (!string.IsNullOrWhiteSpace(outputString))
    {
        sb.AppendLine("Output:");
        sb.AppendLine(outputString);
    }

    return isTimeout
        ? new TimeoutException(sb.ToString())
        : new InvalidOperationException(sb.ToString());
}
```

**?? Informative Errors:**
- **Includes stdout** (pode ter warnings)
- **Includes stderr** (error messages)
- **Distinguishes timeout** vs other failures
- **Helps debugging** startup issues

---

### 2. Process Death Notification

```csharp
public event EventHandler<ServerProcessExitedEventArgs>? ServerProcessExited;

// In RunServer()
process.Exited += (sender, e) => ServerProcessExited?.Invoke(sender, new ServerProcessExitedEventArgs());
```

**Use Cases:**
- **Monitoring:** App pode reagir a crash do servidor
- **Auto-restart:** Implementar retry logic
- **Logging:** Registrar quando servidor morre inesperadamente

---

### 3. Concurrent Restart Protection

```csharp
public async Task RestartServerAsync()
{
    var existingServerTask = _serverTask;
    
    // ... shutdown existing server ...

    // Atomic swap - prevents concurrent restarts
    if (Interlocked.CompareExchange(ref _serverTask, null, existingServerTask) != existingServerTask)
        throw new InvalidOperationException("Server changed while restarting. Concurrent RestartServerAsync()?");

    await StartServerInternalAsync().ConfigureAwait(false);
}
```

**?? Interlocked.CompareExchange Pattern:**
- **Atomic operation** (thread-safe)
- **Detects concurrent** restarts
- **Fails fast** se race condition detectada

---

## ?? Exemplo de Uso Completo

### Basic Usage

```csharp
using Raven.Embedded;

public class Program
{
    public static void Main()
    {
        // Start embedded server (singleton)
        EmbeddedServer.Instance.StartServer();

        try
        {
            // Get document store for database "MyApp"
            using (var store = EmbeddedServer.Instance.GetDocumentStore("MyApp"))
            {
                // Use normally
                using (var session = store.OpenSession())
                {
                    var user = new User { Name = "John" };
                    session.Store(user);
                    session.SaveChanges();
                }

                using (var session = store.OpenSession())
                {
                    var user = session.Query<User>()
                        .FirstOrDefault(u => u.Name == "John");
                    
                    Console.WriteLine($"Loaded: {user.Name}");
                }
            }
        }
        finally
        {
            // Graceful shutdown on app exit
            EmbeddedServer.Instance.Dispose();
        }
    }
}
```

---

### Advanced Configuration

```csharp
// Custom server options
var serverOptions = new ServerOptions
{
    DataDirectory = @"C:\MyApp\Data",
    LogsPath = @"C:\MyApp\Logs",
    ServerDirectory = @"C:\MyApp\RavenDB",
    FrameworkVersion = "8.0.22+", // Wildcard version
    MaxServerStartupTimeDuration = TimeSpan.FromMinutes(2),
    GracefulShutdownTimeout = TimeSpan.FromSeconds(60)
};

// Licensing
serverOptions.Licensing = new ServerOptions.LicensingOptions
{
    LicensePath = @"C:\MyApp\license.json",
    EulaAccepted = true
};

EmbeddedServer.Instance.StartServer(serverOptions);

// Custom database options
var dbOptions = new DatabaseOptions("MyApp")
{
    Conventions = new DocumentConventions
    {
        SaveEnumsAsIntegers = true,
        MaxNumberOfRequestsPerSession = 100
    },
    SkipCreatingDatabase = false // Auto-create if not exists
};

using (var store = await EmbeddedServer.Instance.GetDocumentStoreAsync(dbOptions))
{
    // Use store
}
```

---

### Secured Embedded Server

```csharp
var serverOptions = new ServerOptions()
    .Secured(
        certificate: @"C:\certs\server.pfx",
        certPassword: "secret123"
    );

serverOptions.Licensing = new ServerOptions.LicensingOptions
{
    License = "your-license-key-here",
    EulaAccepted = true
};

EmbeddedServer.Instance.StartServer(serverOptions);

// Store will use HTTPS automatically
using (var store = EmbeddedServer.Instance.GetDocumentStore("SecureApp"))
{
    // All communication is encrypted
}
```

---

### Process Monitoring

```csharp
EmbeddedServer.Instance.ServerProcessExited += (sender, args) =>
{
    Console.WriteLine("?? Server process died unexpectedly!");
    
    // Could implement auto-restart here
    // EmbeddedServer.Instance.RestartServerAsync();
};

EmbeddedServer.Instance.StartServer();

// Get process ID for monitoring
var pid = await EmbeddedServer.Instance.GetServerProcessIdAsync();
Console.WriteLine($"Server running with PID: {pid}");
```

---

### Opening Studio in Browser

```csharp
EmbeddedServer.Instance.StartServer();

// Opens Studio in default browser
EmbeddedServer.Instance.OpenStudioInBrowser();

// Equivalent to navigating to:
// http://127.0.0.1:{dynamicPort}/studio/index.html?disableAnalytics=true
```

---

## ?? Quando Usar Raven.Embedded

### ? Use Cases Ideais

1. **Desktop Applications:**
   ```csharp
   // WPF, WinForms, Avalonia apps
   public partial class App : Application
   {
       protected override void OnStartup(StartupEventArgs e)
       {
           EmbeddedServer.Instance.StartServer();
           base.OnStartup(e);
       }

       protected override void OnExit(ExitEventArgs e)
       {
           EmbeddedServer.Instance.Dispose();
           base.OnExit(e);
       }
   }
   ```

2. **Console Applications:**
   ```csharp
   // CLI tools, batch processors
   static async Task Main()
   {
       EmbeddedServer.Instance.StartServer();
       
       using var store = EmbeddedServer.Instance.GetDocumentStore("ToolDB");
       
       // Process data
       
       EmbeddedServer.Instance.Dispose();
   }
   ```

3. **Integration Tests:**
   ```csharp
   public class DatabaseTests : IDisposable
   {
       private readonly IDocumentStore _store;

       public DatabaseTests()
       {
           EmbeddedServer.Instance.StartServer();
           _store = EmbeddedServer.Instance.GetDocumentStore($"TestDB_{Guid.NewGuid()}");
       }

       [Fact]
       public void Test_Something()
       {
           using (var session = _store.OpenSession())
           {
               // Test code
           }
       }

       public void Dispose()
       {
           _store?.Dispose();
       }
   }
   ```

4. **Offline-First Apps:**
   - Apps que precisam funcionar sem internet
   - Local data storage com sync posterior
   - Mobile/field applications

---

### ? Quando NÃO Usar

1. **Production Web Apps:**
   - Use servidor standalone (melhor performance)
   - Embedded adiciona overhead de process isolation
   - Menos controle sobre recursos

2. **High-Load Scenarios:**
   - Embedded não otimizado para alta concorrência
   - Process overhead pode ser significativo
   - Melhor usar servidor dedicado

3. **Multi-Tenant SaaS:**
   - Precisa de isolamento entre tenants
   - Embedded = single server process
   - Use múltiplos servidores standalone

4. **Clustered Deployments:**
   - Embedded = single node only
   - Para HA/clustering, use servidores standalone

---

## ?? Comparação: Embedded vs Standalone

| Aspecto | Embedded | Standalone |
|---------|----------|------------|
| **Deployment** | ? NuGet package | ? Separate install |
| **Configuration** | ? Code-based | ? Config files |
| **Updates** | ? Via NuGet | ? Manual upgrade |
| **Performance** | ?? Process overhead | ? Native performance |
| **Clustering** | ? Single node | ? Multi-node |
| **Resource Control** | ?? Limited | ? Full control |
| **Debugging** | ?? Attach to process | ? Same debugger |
| **Use Case** | Desktop/Tests | Production/Web |

---

## ?? Lições Aprendidas

### 1. ? Best Practices

**Server Lifecycle:**
```csharp
// ? CORRETO - Start once, reuse stores
EmbeddedServer.Instance.StartServer();
var store1 = EmbeddedServer.Instance.GetDocumentStore("DB1");
var store2 = EmbeddedServer.Instance.GetDocumentStore("DB2");

// ? ERRADO - Multiple starts
EmbeddedServer.Instance.StartServer();
EmbeddedServer.Instance.StartServer(); // Throws exception!
```

**Disposal:**
```csharp
// ? CORRETO - Dispose stores, then server
store.Dispose();
EmbeddedServer.Instance.Dispose();

// ?? ACEITÁVEL - Dispose apenas server (auto-dispose stores)
EmbeddedServer.Instance.Dispose();
```

**Async Startup:**
```csharp
// ? CORRETO - StartServer() é sync mas lança async task
EmbeddedServer.Instance.StartServer();
var store = await EmbeddedServer.Instance.GetDocumentStoreAsync("DB");

// ? ERRADO - Não await StartServer (não existe async version)
await EmbeddedServer.Instance.StartServer(); // Compilation error
```

---

### 2. ?? Pitfalls Comuns

**Framework Version Mismatch:**
```csharp
// ? PROBLEMA: App em .NET 8, mas FrameworkVersion = "6.0.x"
var options = new ServerOptions
{
    FrameworkVersion = "6.0.x" // Won't work if only .NET 8 installed
};

// ? SOLUÇÃO: Match app framework ou use wildcard
var options = new ServerOptions
{
    FrameworkVersion = "8.0.x" // Matches app runtime
};
```

**Data Directory Persistence:**
```csharp
// ? PROBLEMA: Data em temp folder (deletado ao reiniciar)
var options = new ServerOptions
{
    DataDirectory = Path.GetTempPath() // BAD!
};

// ? SOLUÇÃO: Use local app data folder
var options = new ServerOptions
{
    DataDirectory = Path.Combine(
        Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
        "MyApp", "RavenDB")
};
```

**Firewall Issues:**
```csharp
// ?? CUIDADO: Dynamic port pode ser bloqueado por firewall
ServerUrl = "http://127.0.0.1:0" // Random port

// Se precisa porta fixa (para firewall rules)
ServerUrl = "http://127.0.0.1:8080"
```

---

### 3. ?? Performance Tips

**Reuse Document Stores:**
```csharp
// ? CORRETO - Cache store
private static IDocumentStore _store;

public static IDocumentStore GetStore()
{
    return _store ??= EmbeddedServer.Instance.GetDocumentStore("MyApp");
}

// ? ERRADO - Create new store every time
public static IDocumentStore GetStore()
{
    return EmbeddedServer.Instance.GetDocumentStore("MyApp"); // Expensive!
}
```

**Disable Topology Cache:**
```csharp
// JÁ FEITO AUTOMATICAMENTE pelo Embedded!
store.Conventions.DisableTopologyCache = true;

// Não precisa fazer manualmente
```

---

## ?? Métricas & Observações

### Startup Time

```
Cold start (first run): ~2-5 seconds
Warm start (subsequent): ~1-2 seconds
With FrameworkVersion matching: +0.5-1s (dotnet --info call)
```

### Memory Overhead

```
Embedded process: ~50-100 MB base
+ Your data/indexes
+ .NET runtime overhead

vs Standalone: Similar memory, but separate process
```

### Performance Impact

```
HTTP communication overhead: ~0.1-1ms per request (local loopback)
vs In-Process: Zero overhead, mas Embedded usa out-of-process por design
```

---

## ?? Resumo Executivo

### Pontos Fortes

1. **? Zero Configuration:** NuGet install ? código ? funciona
2. **? Process Isolation:** Crash do servidor não mata app
3. **? Lifecycle Management:** Auto-cleanup, graceful shutdown
4. **? Framework Version Matching:** Compatibilidade automática
5. **? Security Support:** HTTPS embedded server
6. **? Testing-Friendly:** Perfeito para integration tests

### Limitações

1. **? Single Node Only:** Sem clustering
2. **? Process Overhead:** HTTP loopback communication
3. **? Limited Control:** Menos tunning options que standalone
4. **?? Deployment Size:** Precisa bundle do servidor (~50MB)

### Decisão: Embedded vs Standalone

**Use Embedded se:**
- Desktop application
- Integration tests
- Offline-first scenarios
- Simplicidade > Performance máxima

**Use Standalone se:**
- Production web apps
- High-load scenarios
- Clustering necessário
- Controle total de recursos

---

## ?? Próximos Passos

Com **Raven.Embedded** concluído, temos agora **7 de 16 análises**:

- ? Sparrow
- ? Voron
- ? Sparrow.Server
- ? Corax
- ? Raven.Server
- ? Raven.Client
- ? **Raven.Embedded** ? NOVO!

**Próximo:** Raven.TestDriver (testing infrastructure) ou iniciar análises temáticas?

---

**Conclusão:** Raven.Embedded é uma abstração elegante que combina simplicidade de uso (NuGet package) com robustez (out-of-process server). É ideal para desktop apps e testes, usando padrões como Singleton, Lazy Initialization e Process Wrapper para gerenciar o lifecycle do servidor de forma transparente.
