# Raven.TestDriver - Testing Infrastructure

## ?? Visão Geral

O **Raven.TestDriver** é uma biblioteca de testing que simplifica drasticamente a escrita de testes de integração para RavenDB. É uma abstração sobre o **Raven.Embedded** que fornece convenções, helpers e lifecycle management automático para testes.

### Propósito

- **Facilitar testes de integração** com RavenDB
- **Automatizar lifecycle** de servers e databases
- **Fornecer helpers** para cenários comuns de teste
- **Isolar testes** (cada teste tem seu próprio database)
- **Simplificar debugging** com Studio integration

### Características Principais

```
Multi-Target: .NET 6.0, .NET 7.0, .NET 8.0, .NET Standard 2.0, .NET Framework 4.6.2
Arquivos: ~4 arquivos .cs
LOC: ~500 linhas de código
Base class pattern para test fixtures
Global + Scoped server support
```

---

## ??? Arquitetura Interna

### Design Pattern: Test Fixture Base Class

```
???????????????????????????????????????????????????
?         RavenTestDriver (Base Class)            ?
?  - Global Server (shared across tests)          ?
?  - Scoped Server (per test class)               ?
?  - Document Store Factory                       ?
?  - Lifecycle Hooks                               ?
???????????????????????????????????????????????????
               ?
    ?????????????????????????
    ?                       ?
????????????????   ????????????????????????
?EmbeddedServer?   ?  DocumentStore per   ?
?(Global/Scoped?   ?  Test Method         ?
????????????????   ????????????????????????
```

### Componentes Principais

1. **RavenTestDriver** - Base class para test fixtures
2. **Global Server** - Singleton server compartilhado entre testes
3. **Scoped Server** - Server isolado por test class
4. **GetDocumentStoreOptions** - Configuração por teste
5. **TestServerOptions** - Configuração do servidor embedded

---

## ?? Principais Funcionalidades

### 1. Base Class Pattern

```csharp
// src/Raven.TestDriver/RavenTestDriver.cs
public class RavenTestDriver : IDisposable
{
    // Global server - shared across ALL test classes
    private static readonly EmbeddedServer GlobalServer = new();
    private static ServerOptions GlobalServerOptions;
    private static readonly Lazy<IDocumentStore> GlobalDocumentStore = 
        new(CreateGlobalDocumentStore, LazyThreadSafetyMode.ExecutionAndPublication);

    // Scoped server - isolated per test class
    private EmbeddedServer _scopedServer;
    private ServerOptions _scopedServerOptions;
    private readonly Lazy<IDocumentStore> _scopedDocumentStore;

    // Track all document stores created by this test driver
    private readonly ConcurrentDictionary<DocumentStore, object> _documentStores = 
        new ConcurrentDictionary<DocumentStore, object>();

    // Unique database name generator
    private static int _index;
}
```

**?? Two-Tier Architecture:**

| Tier | Scope | Lifetime | Use Case |
|------|-------|----------|----------|
| **Global Server** | All test classes | Process lifetime | Fast test execution |
| **Scoped Server** | Single test class | Test class lifetime | Isolation, custom config |

---

### 2. GetDocumentStore - Auto Database Management

```csharp
protected internal IDocumentStore GetDocumentStore(
    GetDocumentStoreOptions options = null, 
    [CallerMemberName] string database = null)
{
    options = options ?? GetDocumentStoreOptions.Default;

    // Auto-generate database name from test method name
    if (string.Equals(database, ".ctor", StringComparison.OrdinalIgnoreCase))
        database = $"{GetType().Name}_ctor";
    else if (string.Equals(database, ".cctor", StringComparison.OrdinalIgnoreCase))
        database = $"{GetType().Name}_cctor";

    var name = database + "_" + Interlocked.Increment(ref _index);
    
    // Get global or scoped document store
    var documentStore = _scopedServer != null 
        ? _scopedDocumentStore.Value 
        : GlobalDocumentStore.Value;

    // Create database record
    var databaseRecord = new DatabaseRecord(name);
    PreConfigureDatabase(databaseRecord); // Hook for subclasses

    // Create database on server
    documentStore.Maintenance.Server.Send(new CreateDatabaseOperation(databaseRecord));

    // Create client document store
    var store = new DocumentStore
    {
        Urls = documentStore.Urls,
        Database = name,
        Conventions =
        {
            DisableTopologyCache = true // Local server, no topology updates needed
        }
    };

    PreInitialize(store); // Hook for subclasses
    store.Initialize();

    // Auto-cleanup on dispose
    store.AfterDispose += (sender, args) =>
    {
        if (_documentStores.TryRemove(store, out _) == false)
            return;

        try
        {
            // Hard delete database when store is disposed
            store.Maintenance.Server.Send(
                new DeleteDatabasesOperation(databaseRecord.DatabaseName, hardDelete: true));
        }
        catch (DatabaseDoesNotExistException) { }
        catch (NoLeaderException) { }
    };

    // Import dump if configured
    AsyncHelpers.RunSync(() => ImportDatabaseAsync(store, name));

    SetupDatabase(store); // Hook for subclasses

    // Wait for indexing if requested
    if (options.WaitForIndexingTimeout.HasValue)
        WaitForIndexing(store, name, options.WaitForIndexingTimeout);

    // Track store for cleanup
    _documentStores[store] = null;

    return store;
}
```

**?? Key Features:**

1. **CallerMemberName Attribute:**
   - Database name = test method name automaticamente
   - Facilita identificação de qual teste criou qual database

2. **Auto-increment Suffix:**
   - `TestMethod_1`, `TestMethod_2`, etc.
   - Permite múltiplos `GetDocumentStore()` no mesmo teste

3. **Lifecycle Hooks:**
   - `PreConfigureDatabase(DatabaseRecord)` - Antes de criar database
   - `PreInitialize(IDocumentStore)` - Antes de inicializar store
   - `SetupDatabase(IDocumentStore)` - Depois de inicializar store

4. **Auto-Cleanup:**
   - `store.AfterDispose` hook deleta database automaticamente
   - Hard delete (não fica lixo no servidor)

---

### 3. Global vs Scoped Server

#### Global Server (Default)

```csharp
private static IDocumentStore CreateGlobalDocumentStore()
{
    var options = GlobalServerOptions ?? new TestServerOptions();
    
    // Empty config file (RunInMemory mode)
    options.CommandLineArgs.Insert(0, 
        $"-c {CommandLineArgumentEscaper.EscapeSingleArg(EmptySettingsFile.FullName)}");
    options.CommandLineArgs.Add("--RunInMemory=true");

    GlobalServer.StartServer(options);

    var url = AsyncHelpers.RunSync(() => GlobalServer.GetServerUriAsync());

    var store = new DocumentStore
    {
        Urls = new[] { url.AbsoluteUri },
        Conventions = { DisableTopologyCache = true }
    };

    store.Initialize();
    return store;
}
```

**Configure Global Server:**

```csharp
[SetUpFixture] // NUnit
public class GlobalSetup
{
    [OneTimeSetUp]
    public void Setup()
    {
        RavenTestDriver.ConfigureServer(new TestServerOptions
        {
            DataDirectory = @"C:\Temp\RavenTests",
            FrameworkVersion = "8.0.x"
        });
    }
}

// Xunit equivalent
public class AssemblyFixture : IDisposable
{
    public AssemblyFixture()
    {
        RavenTestDriver.ConfigureServer(new TestServerOptions { ... });
    }
}
```

#### Scoped Server (Per Test Class)

```csharp
public class MyTests : RavenTestDriver, IDisposable
{
    private readonly IDisposable _scopedServerDisposable;

    public MyTests()
    {
        _scopedServerDisposable = ConfigureScopedServer(new TestServerOptions
        {
            // Custom settings for this test class only
            DataDirectory = @"C:\Temp\MyTests"
        });
    }

    public new void Dispose()
    {
        _scopedServerDisposable?.Dispose();
        base.Dispose();
    }
}
```

**?? Comparison:**

| Aspect | Global Server | Scoped Server |
|--------|---------------|---------------|
| **Startup Cost** | ? Once per process | ?? Once per test class |
| **Isolation** | ?? Shared resources | ? Complete isolation |
| **Custom Config** | ? Limited (must configure before first test) | ? Per test class |
| **Performance** | ? Fastest | ?? Slower startup |
| **Use Case** | Most tests | Special config needs |

---

### 4. WaitForIndexing - Test Synchronization

```csharp
protected void WaitForIndexing(
    IDocumentStore store, 
    string database = null, 
    TimeSpan? timeout = null)
{
    var admin = store.Maintenance.ForDatabase(database);
    timeout = timeout ?? TimeSpan.FromMinutes(1);

    var sp = Stopwatch.StartNew();
    while (sp.Elapsed < timeout.Value)
    {
        var statistics = admin.Send(new GetStatisticsOperation());
        var indexes = statistics.Indexes.Where(x => x.State != IndexState.Disabled);

        // Check if all indexes are non-stale
        if (indexes.All(x => x.IsStale == false
            && x.Name.StartsWith(Constants.Documents.Indexing.SideBySideIndexNamePrefix) == false))
            return; // Success!

        // Check for index errors
        if (statistics.Indexes.Any(x => x.State == IndexState.Error))
            break;

        Thread.Sleep(100);
    }

    // Timeout or error - get detailed error info
    var errors = admin.Send(new GetIndexErrorsOperation());

    string allIndexErrorsText = string.Empty;
    if (errors != null && errors.Length > 0)
    {
        var errorDetails = string.Join("\r\n",
            errors.Select(FormatIndexErrors));
        allIndexErrorsText = $"Indexing errors:\r\n{errorDetails}";
    }

    throw new TimeoutException(
        $"Indexes stayed stale for more than {timeout.Value}.{allIndexErrorsText}");
}
```

**?? Usage:**

```csharp
[Test]
public void Test_Query_After_Insert()
{
    using (var store = GetDocumentStore())
    {
        new UsersByName().Execute(store); // Create index

        using (var session = store.OpenSession())
        {
            session.Store(new User { Name = "John" });
            session.SaveChanges();
        }

        // Wait for index to catch up
        WaitForIndexing(store, timeout: TimeSpan.FromSeconds(30));

        using (var session = store.OpenSession())
        {
            var users = session.Query<User, UsersByName>()
                .Where(u => u.Name == "John")
                .ToList();

            Assert.Equal(1, users.Count); // Guaranteed to pass
        }
    }
}
```

**?? Auto Wait in GetDocumentStore:**

```csharp
var store = GetDocumentStore(new GetDocumentStoreOptions
{
    WaitForIndexingTimeout = TimeSpan.FromSeconds(30)
});
// Automatically waits for indexing after SetupDatabase() hook
```

---

### 5. Database Import Support

```csharp
protected virtual string DatabaseDumpFilePath => null;
protected virtual Stream DatabaseDumpFileStream => null;

private async Task ImportDatabaseAsync(
    DocumentStore docStore, 
    string database, 
    TimeSpan? timeout = null)
{
    var options = new DatabaseSmugglerImportOptions();
    
    if (DatabaseDumpFilePath != null)
    {
        var operation = await docStore.Smuggler
            .ForDatabase(database)
            .ImportAsync(options, DatabaseDumpFilePath);
        await operation.WaitForCompletionAsync(timeout);
    }
    else if (DatabaseDumpFileStream != null)
    {
        var operation = await docStore.Smuggler
            .ForDatabase(database)
            .ImportAsync(options, DatabaseDumpFileStream);
        await operation.WaitForCompletionAsync(timeout);
    }
}
```

**Usage:**

```csharp
public class MyTests : RavenTestDriver
{
    protected override string DatabaseDumpFilePath => @"C:\TestData\backup.ravendbdump";

    [Test]
    public void Test_Against_Production_Data()
    {
        using (var store = GetDocumentStore())
        {
            // Database is automatically imported from dump file
            
            using (var session = store.OpenSession())
            {
                var user = session.Load<User>("users/1");
                Assert.NotNull(user);
            }
        }
    }
}
```

---

### 6. WaitForUserToContinueTheTest - Debug Helper

```csharp
protected void WaitForUserToContinueTheTest(IDocumentStore store)
{
    if (Debugger.IsAttached == false)
        return; // Only works when debugging

    var databaseNameEncoded = Uri.EscapeDataString(store.Database);
    var documentsPage = store.Urls[0] + 
        "/studio/index.html#databases/documents?&database=" + 
        databaseNameEncoded + "&withStop=true";

    OpenBrowser(documentsPage); // Opens Studio in browser

    // Wait for user to create "Debug/Done" document
    do
    {
        Thread.Sleep(500);

        using (var session = store.OpenSession())
        {
            if (session.Advanced.Exists("Debug/Done"))
            {
                session.Delete("Debug/Done");
                session.SaveChanges();
                break; // Continue test
            }
        }
    } while (true);
}
```

**Usage:**

```csharp
[Test]
public void Test_Complex_Query()
{
    using (var store = GetDocumentStore())
    {
        // ... setup data ...

        WaitForUserToContinueTheTest(store);
        
        // Test pauses here, Studio opens in browser
        // User can inspect data, run queries, etc.
        // Create document "Debug/Done" to continue

        // ... assertions ...
    }
}
```

**?? Workflow:**
1. Test hits `WaitForUserToContinueTheTest()`
2. Studio opens automatically in browser
3. Developer inspects data, tests queries manually
4. Create document with ID `Debug/Done`
5. Test resumes and continues assertions

---

### 7. Lifecycle Hooks Pattern

```csharp
public class MyTests : RavenTestDriver
{
    // Hook 1: Configure database before creation
    protected override void PreConfigureDatabase(DatabaseRecord databaseRecord)
    {
        databaseRecord.Settings = new Dictionary<string, string>
        {
            ["Indexing.MaxTimeForDocumentTransactionToRemainOpen"] = "60"
        };
    }

    // Hook 2: Configure store before initialization
    protected override void PreInitialize(IDocumentStore documentStore)
    {
        documentStore.Conventions.MaxNumberOfRequestsPerSession = 100;
        documentStore.Conventions.SaveEnumsAsIntegers = true;
    }

    // Hook 3: Setup database after initialization
    protected override void SetupDatabase(IDocumentStore documentStore)
    {
        // Create indexes
        new UsersByName().Execute(documentStore);
        new ProductsByCategory().Execute(documentStore);

        // Seed data
        using (var session = documentStore.OpenSession())
        {
            session.Store(new User { Name = "Admin" });
            session.SaveChanges();
        }
    }
}
```

**?? Hook Order:**

```
GetDocumentStore()
  ?
  ??? PreConfigureDatabase(DatabaseRecord)
  ?     ??? Modify database settings
  ?
  ??? CreateDatabaseOperation
  ?
  ??? PreInitialize(IDocumentStore)
  ?     ??? Configure conventions
  ?
  ??? store.Initialize()
  ?
  ??? ImportDatabaseAsync()
  ?     ??? Import dump if configured
  ?
  ??? SetupDatabase(IDocumentStore)
  ?     ??? Create indexes, seed data
  ?
  ??? WaitForIndexing() (if configured)
```

---

## ?? Padrões de Teste

### Pattern 1: Simple Test

```csharp
public class UserTests : RavenTestDriver
{
    [Fact]
    public void Can_Store_And_Load_User()
    {
        using (var store = GetDocumentStore())
        using (var session = store.OpenSession())
        {
            // Arrange
            var user = new User { Name = "John", Age = 30 };

            // Act
            session.Store(user);
            session.SaveChanges();

            // Assert
            var loaded = session.Load<User>(user.Id);
            Assert.Equal("John", loaded.Name);
            Assert.Equal(30, loaded.Age);
        }
    }
}
```

---

### Pattern 2: Query Test with WaitForIndexing

```csharp
public class UserQueryTests : RavenTestDriver
{
    protected override void SetupDatabase(IDocumentStore store)
    {
        // Create index
        new Users_ByName().Execute(store);
    }

    [Fact]
    public void Can_Query_Users_By_Name()
    {
        using (var store = GetDocumentStore(new GetDocumentStoreOptions
        {
            WaitForIndexingTimeout = TimeSpan.FromSeconds(30)
        }))
        using (var session = store.OpenSession())
        {
            // Arrange
            session.Store(new User { Name = "John" });
            session.Store(new User { Name = "Jane" });
            session.SaveChanges();

            // Indexing is automatically awaited

            // Act
            var johns = session.Query<User, Users_ByName>()
                .Where(u => u.Name == "John")
                .ToList();

            // Assert
            Assert.Single(johns);
            Assert.Equal("John", johns[0].Name);
        }
    }
}
```

---

### Pattern 3: Test with Seed Data

```csharp
public class ProductTests : RavenTestDriver
{
    protected override void SetupDatabase(IDocumentStore store)
    {
        using (var session = store.OpenSession())
        {
            // Seed common data for all tests
            session.Store(new Category { Id = "categories/1", Name = "Electronics" });
            session.Store(new Category { Id = "categories/2", Name = "Books" });
            session.SaveChanges();
        }
    }

    [Fact]
    public void Can_Load_Seeded_Categories()
    {
        using (var store = GetDocumentStore())
        using (var session = store.OpenSession())
        {
            var category = session.Load<Category>("categories/1");
            Assert.Equal("Electronics", category.Name);
        }
    }
}
```

---

### Pattern 4: Test with Production Dump

```csharp
public class ProductionDataTests : RavenTestDriver
{
    protected override string DatabaseDumpFilePath => @"C:\Dumps\production-2024.ravendbdump";

    [Fact]
    public void Can_Query_Production_Data()
    {
        using (var store = GetDocumentStore())
        using (var session = store.OpenSession())
        {
            // Database already has production data imported
            var users = session.Query<User>().ToList();
            Assert.True(users.Count > 1000);
        }
    }
}
```

---

### Pattern 5: Custom Server Configuration

```csharp
public class HighMemoryTests : RavenTestDriver
{
    private readonly IDisposable _scopedServer;

    public HighMemoryTests()
    {
        var options = new TestServerOptions();
        options.CommandLineArgs.Add("--Memory.LowMemoryLimit=2048");
        options.CommandLineArgs.Add("--Memory.HighMemoryLimit=8192");

        _scopedServer = ConfigureScopedServer(options);
    }

    [Fact]
    public void Test_With_High_Memory()
    {
        using (var store = GetDocumentStore())
        {
            // Server has custom memory settings
        }
    }

    public new void Dispose()
    {
        _scopedServer?.Dispose();
        base.Dispose();
    }
}
```

---

### Pattern 6: Fiddler Integration (Network Debugging)

```csharp
public class NetworkTests : RavenTestDriver
{
    public NetworkTests()
    {
        // Configure server to allow external access (for Fiddler)
        ConfigureScopedServer(TestServerOptions.UseFiddler());
    }

    [Fact]
    public void Test_With_Fiddler_Capture()
    {
        using (var store = GetDocumentStore())
        {
            // All HTTP traffic can be captured by Fiddler
            // Server URL: http://{MachineName}:port (not 127.0.0.1)
        }
    }
}
```

**TestServerOptions.UseFiddler():**

```csharp
public static TestServerOptions UseFiddler()
{
    return new TestServerOptions
    {
        ServerUrl = $"http://{Environment.MachineName}:0",
        CommandLineArgs =
        {
            "--Security.UnsecuredAccessAllowed=PrivateNetwork"
        }
    };
}
```

**?? Por que MachineName?**
- Fiddler não captura tráfego para `127.0.0.1` ou `localhost`
- Usar `{MachineName}` força tráfego pela network stack
- Permite Fiddler capturar requests HTTP

---

## ?? Técnicas de Implementação

### 1. CallerMemberName Magic

```csharp
protected internal IDocumentStore GetDocumentStore(
    GetDocumentStoreOptions options = null, 
    [CallerMemberName] string database = null) // ? MAGIC!
{
    // database parameter is automatically filled with caller method name!
}

// Usage:
[Test]
public void MyTest()
{
    var store = GetDocumentStore(); // database = "MyTest"
}
```

**Benefits:**
- **Zero boilerplate** (não precisa passar nome)
- **Readable database names** (fácil identificar no Studio)
- **Automatic uniqueness** (cada teste tem database único)

---

### 2. Lazy Singleton Pattern

```csharp
// Thread-safe, lazy initialization
private static readonly Lazy<IDocumentStore> GlobalDocumentStore = 
    new(CreateGlobalDocumentStore, LazyThreadSafetyMode.ExecutionAndPublication);

// First access triggers initialization
var store = GlobalDocumentStore.Value;
```

**Thread-Safety Modes:**
- `ExecutionAndPublication`: Thread-safe, exception cached
- Alternative: `PublicationOnly` (exception não cached)

---

### 3. AfterDispose Hook Pattern

```csharp
store.AfterDispose += (sender, args) =>
{
    // Cleanup logic here
    store.Maintenance.Server.Send(
        new DeleteDatabasesOperation(name, hardDelete: true));
};
```

**?? Why AfterDispose instead of BeforeDispose?**
- **Store is still usable** em AfterDispose
- **Can call Maintenance** operations
- **Guaranteed execution** (even if Dispose() throws)

---

### 4. Empty Settings File Trick

```csharp
private static FileInfo EmptySettingsFile
{
    get
    {
        if (_emptySettingsFile == null)
        {
            _emptySettingsFile = new FileInfo(Path.GetTempFileName());
            File.WriteAllText(_emptySettingsFile.FullName, "{}");
        }
        return _emptySettingsFile;
    }
}

// Usage
options.CommandLineArgs.Insert(0, 
    $"-c {CommandLineArgumentEscaper.EscapeSingleArg(EmptySettingsFile.FullName)}");
```

**?? Por que?**
- **Evita carregar** settings do sistema
- **Clean slate** para testes
- **RunInMemory mode** fica mais confiável

---

### 5. Interlocked Counter for Uniqueness

```csharp
private static int _index;

var name = database + "_" + Interlocked.Increment(ref _index);
// Example: "MyTest_1", "MyTest_2", "MyTest_3"
```

**Thread-Safe Increment:**
- `Interlocked.Increment()` é atomic
- Safe para multi-threaded test runners (xUnit, NUnit parallel)

---

## ?? Comparação: RavenTestDriver vs Alternatives

### vs Manual Embedded Server

| Aspect | RavenTestDriver | Manual Embedded |
|--------|-----------------|-----------------|
| **Setup Code** | ? Minimal (inherit base class) | ? Verbose (manual lifecycle) |
| **Database Cleanup** | ? Automatic | ? Manual |
| **Unique Databases** | ? Auto-generated names | ? Manual naming |
| **Lifecycle Hooks** | ? Virtual methods | ? Custom implementation |
| **WaitForIndexing** | ? Built-in | ? Custom wait logic |

---

### vs In-Memory Databases (e.g., SQLite)

| Aspect | RavenDB TestDriver | SQLite In-Memory |
|--------|-------------------|------------------|
| **Realism** | ? Real RavenDB server | ?? Different engine |
| **Features** | ? All features (indexes, queries) | ? Limited SQL |
| **Performance** | ?? Slower (HTTP + indexing) | ? Very fast |
| **Isolation** | ? Per-database | ? Per-connection |

**Recommendation:** Use RavenTestDriver for **integration tests**, in-memory for **unit tests**

---

## ?? Exemplo Completo: Test Suite

```csharp
using Xunit;
using Raven.TestDriver;
using Raven.Client.Documents.Indexes;

namespace MyApp.Tests
{
    // Test class inherits from RavenTestDriver
    public class UserTests : RavenTestDriver
    {
        // Constructor - optional custom configuration
        public UserTests()
        {
            // Optional: Use scoped server with custom settings
            // ConfigureScopedServer(new TestServerOptions { ... });
        }

        // Hook: Configure database before creation
        protected override void PreConfigureDatabase(DatabaseRecord databaseRecord)
        {
            databaseRecord.Settings = new Dictionary<string, string>
            {
                ["Indexing.MapTimeout"] = "60"
            };
        }

        // Hook: Setup indexes and seed data
        protected override void SetupDatabase(IDocumentStore store)
        {
            // Create indexes
            new Users_ByName().Execute(store);

            // Seed common data
            using (var session = store.OpenSession())
            {
                session.Store(new User { Name = "Admin", Email = "admin@test.com" });
                session.SaveChanges();
            }
        }

        [Fact]
        public void Can_Store_User()
        {
            using (var store = GetDocumentStore()) // Auto database name: "Can_Store_User_1"
            using (var session = store.OpenSession())
            {
                var user = new User { Name = "John", Email = "john@test.com" };
                session.Store(user);
                session.SaveChanges();

                Assert.NotNull(user.Id);
            }
            // Database automatically deleted on dispose
        }

        [Fact]
        public void Can_Query_Users()
        {
            using (var store = GetDocumentStore(new GetDocumentStoreOptions
            {
                WaitForIndexingTimeout = TimeSpan.FromSeconds(30) // Auto-wait for indexes
            }))
            using (var session = store.OpenSession())
            {
                // Arrange
                session.Store(new User { Name = "Alice", Email = "alice@test.com" });
                session.Store(new User { Name = "Bob", Email = "bob@test.com" });
                session.SaveChanges();

                // Indexing automatically awaited

                // Act
                var users = session.Query<User, Users_ByName>()
                    .Where(u => u.Name.StartsWith("A"))
                    .ToList();

                // Assert
                Assert.Single(users);
                Assert.Equal("Alice", users[0].Name);
            }
        }

        [Fact]
        public void Can_Load_Seeded_Admin()
        {
            using (var store = GetDocumentStore())
            using (var session = store.OpenSession())
            {
                // Admin user was seeded in SetupDatabase()
                var admin = session.Query<User>()
                    .First(u => u.Name == "Admin");

                Assert.Equal("admin@test.com", admin.Email);
            }
        }

        [Fact(Skip = "Manual debugging only")]
        public void Debug_User_Queries()
        {
            using (var store = GetDocumentStore())
            {
                using (var session = store.OpenSession())
                {
                    session.Store(new User { Name = "Debug", Email = "debug@test.com" });
                    session.SaveChanges();
                }

                // Opens Studio in browser, waits for "Debug/Done" document
                WaitForUserToContinueTheTest(store);

                // Continue with assertions
            }
        }
    }

    // Index definition
    public class Users_ByName : AbstractIndexCreationTask<User>
    {
        public Users_ByName()
        {
            Map = users => from user in users
                           select new { user.Name };
        }
    }

    // Entity
    public class User
    {
        public string Id { get; set; }
        public string Name { get; set; }
        public string Email { get; set; }
    }
}
```

---

## ?? Best Practices

### 1. ? Inherit from RavenTestDriver

```csharp
// ? CORRETO
public class MyTests : RavenTestDriver
{
    [Fact]
    public void Test()
    {
        using (var store = GetDocumentStore())
        {
            // Test code
        }
    }
}

// ? ERRADO - manual setup
public class MyTests
{
    [Fact]
    public void Test()
    {
        var server = new EmbeddedServer();
        server.StartServer();
        // ... manual lifecycle management
    }
}
```

---

### 2. ? Use GetDocumentStore() with Using

```csharp
// ? CORRETO - auto cleanup
[Fact]
public void Test()
{
    using (var store = GetDocumentStore())
    {
        // Test code
    } // Database deleted automatically
}

// ? ERRADO - no cleanup
[Fact]
public void Test()
{
    var store = GetDocumentStore();
    // Test code
    // Database NOT deleted! Memory leak!
}
```

---

### 3. ? Wait for Indexing in Query Tests

```csharp
// ? CORRETO - explicit wait
[Fact]
public void Test_Query()
{
    using (var store = GetDocumentStore(new GetDocumentStoreOptions
    {
        WaitForIndexingTimeout = TimeSpan.FromSeconds(30)
    }))
    {
        // Insert data
        // Query - guaranteed non-stale
    }
}

// ?? ALTERNATIVA - manual wait
[Fact]
public void Test_Query_Manual()
{
    using (var store = GetDocumentStore())
    {
        // Insert data
        WaitForIndexing(store);
        // Query
    }
}

// ? ERRADO - no wait, query may be stale!
[Fact]
public void Test_Query_Wrong()
{
    using (var store = GetDocumentStore())
    {
        session.Store(new User { Name = "John" });
        session.SaveChanges();
        
        var users = session.Query<User>().ToList();
        // May be empty! Indexing async!
    }
}
```

---

### 4. ? Use Scoped Server for Custom Config

```csharp
// ? CORRETO - scoped server para config especial
public class SpecialTests : RavenTestDriver, IDisposable
{
    private readonly IDisposable _scoped;

    public SpecialTests()
    {
        _scoped = ConfigureScopedServer(new TestServerOptions
        {
            // Custom settings
        });
    }

    public new void Dispose()
    {
        _scoped?.Dispose();
        base.Dispose();
    }
}

// ? ERRADO - tentar mudar global server depois de inicializado
public class WrongTests : RavenTestDriver
{
    [Fact]
    public void Test()
    {
        ConfigureServer(new TestServerOptions { ... }); // THROWS if GlobalServer already started!
    }
}
```

---

### 5. ? Dispose RavenTestDriver in Test Cleanup

```csharp
// ? CORRETO - Dispose no cleanup
public class MyTests : RavenTestDriver, IDisposable
{
    [Fact]
    public void Test1() { }

    [Fact]
    public void Test2() { }

    public new void Dispose()
    {
        base.Dispose(); // Cleanup all document stores
    }
}

// Para xUnit, pode usar IClassFixture
public class MyTestsFixture : RavenTestDriver, IDisposable
{
    // Shared across tests
}

public class MyTests : IClassFixture<MyTestsFixture>
{
    private readonly MyTestsFixture _fixture;

    public MyTests(MyTestsFixture fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public void Test()
    {
        using (var store = _fixture.GetDocumentStore())
        {
            // ...
        }
    }
}
```

---

## ?? Performance Tips

### 1. Reuse Global Server

```csharp
// ? FAST - Global server shared
public class Test1 : RavenTestDriver { } // Uses global
public class Test2 : RavenTestDriver { } // Uses global
public class Test3 : RavenTestDriver { } // Uses global

// ?? SLOW - Each class starts new server
public class Test1 : RavenTestDriver 
{
    public Test1() { ConfigureScopedServer(...); }
}
public class Test2 : RavenTestDriver 
{
    public Test2() { ConfigureScopedServer(...); }
}
```

**Metrics:**
- Global server startup: **~2-5s once**
- Scoped server startup: **~2-5s per test class**

---

### 2. Minimize WaitForIndexing Calls

```csharp
// ? EFFICIENT - Wait once per test
[Fact]
public void Test()
{
    using (var store = GetDocumentStore(new GetDocumentStoreOptions
    {
        WaitForIndexingTimeout = TimeSpan.FromSeconds(30)
    }))
    {
        // Setup data
        // All queries use fresh indexes
    }
}

// ? INEFFICIENT - Multiple waits
[Fact]
public void Test()
{
    using (var store = GetDocumentStore())
    {
        // Insert data
        WaitForIndexing(store); // Wait 1
        
        // Insert more data
        WaitForIndexing(store); // Wait 2
        
        // Unnecessary waits!
    }
}
```

---

### 3. Use SetupDatabase for Common Data

```csharp
// ? EFFICIENT - Seed once in SetupDatabase
protected override void SetupDatabase(IDocumentStore store)
{
    using (var session = store.OpenSession())
    {
        for (int i = 0; i < 1000; i++)
            session.Store(new User { Name = $"User{i}" });
        session.SaveChanges();
    }
}

// ? INEFFICIENT - Seed in every test
[Fact]
public void Test1()
{
    using (var store = GetDocumentStore())
    {
        // Insert 1000 users - SLOW!
    }
}

[Fact]
public void Test2()
{
    using (var store = GetDocumentStore())
    {
        // Insert 1000 users again - SLOW!
    }
}
```

---

## ?? Resumo Executivo

### Pontos Fortes

1. **? Minimal Boilerplate:** Inherit base class, call `GetDocumentStore()`
2. **? Auto Lifecycle:** Databases auto-created and auto-deleted
3. **? Isolation:** Each test gets unique database
4. **? Hooks:** Virtual methods para customização
5. **? Debug Support:** `WaitForUserToContinueTheTest()` abre Studio
6. **? CallerMemberName:** Database names automáticos
7. **? Global + Scoped:** Flexibilidade de configuração

### Quando Usar

**? Use RavenTestDriver para:**
- Integration tests
- Feature tests
- E2E tests (com RavenDB)
- Testing indexes, queries
- Testing migrations

**? NÃO use para:**
- Unit tests (too heavy)
- Performance benchmarks (overhead de HTTP)
- Load tests (embedded não é production-like)

### Comparison Summary

| Aspect | TestDriver | Manual Embedded | In-Memory DB |
|--------|-----------|-----------------|--------------|
| **Setup** | ? Minimal | ? Verbose | ? Minimal |
| **Cleanup** | ? Auto | ? Manual | ? Auto |
| **Realism** | ? Real server | ? Real server | ? Different |
| **Speed** | ?? Medium | ?? Medium | ? Fast |
| **Features** | ? All | ? All | ? Limited |

---

## ?? Conclusão

**Raven.TestDriver** é uma abstração **poderosa e elegante** sobre **Raven.Embedded** que elimina boilerplate e fornece convenções sensatas para testes. 

**Key Takeaways:**

1. **Inherit RavenTestDriver** ? Instant test infrastructure
2. **GetDocumentStore()** ? Auto unique databases
3. **CallerMemberName** ? Zero-boilerplate naming
4. **Lifecycle Hooks** ? Customização sem complexity
5. **WaitForIndexing** ? Reliable query tests
6. **Global/Scoped** ? Performance vs Isolation trade-off

**Best for:** Integration tests que precisam de RavenDB **real** com **minimal setup**.

---

**Próximos Passos:** Com **Raven.TestDriver** completo, temos **8 de 16 análises** (50%! ??). Agora podemos:

1. **Iniciar análises temáticas** (Performance Patterns, Memory Management, etc.)
2. **Deep Dive no Raven.Server** (usar o plano já criado)

O que você prefere? ??
