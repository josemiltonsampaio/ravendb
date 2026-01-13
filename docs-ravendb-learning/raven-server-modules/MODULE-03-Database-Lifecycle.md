# Raven.Server - Módulo 3: DocumentDatabase Lifecycle ??

## ?? Visão Geral

O **DocumentDatabase Lifecycle** gerencia todo o ciclo de vida de uma database no RavenDB, desde a inicialização até o graceful shutdown. Este é um dos módulos mais complexos devido à coordenação de dezenas de subsistemas interdependentes.

### Propósito
- Inicializar databases sob demanda (lazy loading)
- Coordenar startup de todos os subsistemas
- Gerenciar unload/idle databases
- Implementar graceful shutdown
- Proteção contra catastrophic failures
- Tracking de uso para idle detection

### Posição na Arquitetura
```
HTTP Request (Módulo 1)
         ?
    RouteInformation.CreateDatabase()
         ?
    DatabasesLandlord.TryGetOrCreateDatabase()
         ?
    DocumentDatabase constructor + Initialize()
         ?
    ???????????????????????????????????????
    ?  DocumentDatabase (Orquestrador)    ?
    ?  ????????????????? ?????????????????
    ?  ? DocumentsStorage ? ? IndexStore   ??
    ?  ????????????????? ?????????????????
    ?  ????????????????? ?????????????????
    ?  ? TxMerger      ? ? Replication  ??
    ?  ????????????????? ?????????????????
    ?  ????????????????? ?????????????????
    ?  ? ETL           ? ? Subscriptions??
    ?  ????????????????? ?????????????????
    ???????????????????????????????????????
         ?
    Usage Tracking ? Idle Detection ? Unload
```

### Estatísticas
```
Arquivo principal: DocumentDatabase.cs (~4500 linhas!)
Arquivos relacionados: 5+ arquivos
Subsistemas coordenados: 20+
Complexidade: ???? ALTA
```

---

## ??? Arquitetura Interna

### 1. **DocumentDatabase** - O Orquestrador Central

**Localização:** `src/Raven.Server/Documents/DocumentDatabase.cs`

```csharp
public class DocumentDatabase : IDisposable
{
    private readonly ServerStore _serverStore;
    private readonly Action<LogLevel, string> _addToInitLog;
    private readonly RavenLogger _logger;
    private readonly DisposeOnce<SingleAttempt> _disposeOnce;
    
    // Shutdown coordination
    private readonly CancellationTokenSource _databaseShutdown;
    public CancellationToken DatabaseShutdown => _databaseShutdown.Token;
    public AsyncManualResetEvent DatabaseShutdownCompleted { get; }
    
    // Usage tracking para idle detection
    private long _usages;
    private readonly ManualResetEventSlim _waitForUsagesOnDisposal;
    private long _preventUnloadCounter;
    
    // Subsistemas principais
    public DocumentsStorage DocumentsStorage { get; private set; }
    public IndexStore IndexStore { get; private set; }
    public DocumentsTransactionOperationsMerger TxMerger { get; }
    public ReplicationLoader ReplicationLoader { get; internal set; }
    public EtlLoader EtlLoader { get; private set; }
    public SubscriptionStorage SubscriptionStorage { get; }
    // ... 15+ subsistemas adicionais
    
    public readonly string Name;
    public RavenConfiguration Configuration { get; }
    public readonly DateTime StartTime;
}
```

**Responsabilidades:**
- Coordenar inicialização de todos os subsistemas
- Gerenciar DatabaseRecord changes
- Tracking de último acesso (idle detection)
- Coordenar graceful shutdown
- Error recovery e catastrophic failure handling

### 2. **DatabasesLandlord** - Gerenciador de Databases

**Localização:** `src/Raven.Server/Documents/DatabasesLandlord.cs`

```csharp
public sealed class DatabasesLandlord : IDisposable
{
    // Cache de databases carregados
    public readonly ResourceCache<DocumentDatabase> DatabasesCache;
    public readonly ResourceCache<ShardedDatabaseContext> ShardedDatabasesCache;
    
    // Idle tracking
    public readonly ConcurrentDictionary<StringSegment, DateTime> LastRecentlyUsed;
    
    // Controle de carga concorrente
    internal SemaphoreSlim _databaseSemaphore;
    internal TimeSpan _concurrentDatabaseLoadTimeout;
    
    public CatastrophicFailureHandler CatastrophicFailureHandler { get; }
    
    public DatabaseSearchResult TryGetOrCreateDatabase(StringSegment databaseName)
    {
        // Verifica se já está carregado
        if (TryGetResourceStore(databaseName, out var databaseTask))
            return new DatabaseSearchResult(Status.Database, databaseTask, null);
        
        // Carrega sob demanda
        return new DatabaseSearchResult(Status.Database, 
            TryGetOrCreateResourceStore(databaseName), null);
    }
}
```

**Responsabilidades:**
- Lazy loading de databases
- Cache de databases ativas
- Unload de databases idle
- Prevenção de sobrecarga (semaphore)
- Coordenação de wakeup timers

### 3. **Catastrophic Failure Handler**

**Localização:** `src/Raven.Server/Documents/CatastrophicFailureHandler.cs`

```csharp
public sealed class CatastrophicFailureHandler
{
    internal TimeSpan TimeToWaitBeforeUnloadingDatabase = TimeSpan.FromSeconds(2);
    internal int MaxDatabaseUnloads = 3;
    internal TimeSpan NoFailurePeriod = TimeSpan.FromMinutes(15);
    
    private readonly ConcurrentDictionary<Guid, FailureStats> _errorsPerEnvironment;
    
    public void Execute(string databaseName, Exception e, Guid environmentId, 
        string path, string stacktrace)
    {
        var stats = _errorsPerEnvironment.GetOrAdd(environmentId, 
            x => FailureStats.Create(MaxDatabaseUnloads));
        
        if (stats.WillUnloadDatabase == false)
        {
            // Já atingiu limite de unloads, não faz mais nada
            if (DateTime.UtcNow - stats.LastUnloadTime > NoFailurePeriod)
            {
                // Resetar após período sem falhas
                stats.NumberOfUnloads = 0;
                stats.LastUnloadTime = DateTime.MinValue;
            }
            else
            {
                return;
            }
        }
        
        stats.DatabaseUnloadTask = Task.Run(async () =>
        {
            // 1. Notificar antes de unload
            _serverStore.NotificationCenter.Add(AlertRaised.Create(...));
            
            // 2. Dar tempo para cliente receber resposta
            await Task.Delay(TimeToWaitBeforeUnloadingDatabase);
            
            // 3. Unload database
            stats.NumberOfUnloads++;
            stats.LastUnloadTime = DateTime.UtcNow;
            (await _databasesLandlord.UnloadAndLockDatabase(databaseName, 
                "CatastrophicFailure"))?.Dispose();
        });
    }
}
```

---

## ?? Fluxo de Inicialização Detalhado

### Fase 1: Constructor - Criação de Componentes

```csharp
public DocumentDatabase(string name, RavenConfiguration configuration, 
    ServerStore serverStore, Action<LogLevel, string> addToInitLog)
{
    Name = name;
    Loggers = new DatabaseLoggersContext(this);
    _logger = Loggers.GetLogger<DocumentDatabase>();
    _serverStore = serverStore;
    _addToInitLog = addToInitLog;
    
    StartTime = Time.GetUtcNow();
    LastAccessTime = Time.GetUtcNow();
    Configuration = configuration;
    
    // CancellationToken linkado ao server shutdown
    _databaseShutdown = CancellationTokenSource.CreateLinkedTokenSource(
        serverStore.ServerShutdown);
    
    _disposeOnce = new DisposeOnce<SingleAttempt>(DisposeInternal);
    
    try
    {
        // 1. File locking (evita múltiplas instâncias)
        if (Configuration.Core.RunInMemory == false)
        {
            _fileLocker = new FileLocker(
                Configuration.Core.DataDirectory.Combine("db.lock").FullPath);
            _fileLocker.TryAcquireWriteLock(_logger);
        }
        
        // 2. Criar subsistemas básicos (ordem importa!)
        Smuggler = new DatabaseSmugglerFactory(this);
        QueryMetadataCache = new QueryMetadataCache();
        IoChanges = new IoChangesNotifications { ... };
        Changes = new DocumentsChanges();
        TombstoneCleaner = new TombstoneCleaner(this);
        
        // 3. Storage layer
        DocumentsStorage = CreateDocumentsStorage(addToInitLog);
        CompareExchangeStorage = new CompareExchangeStorage(this);
        
        // 4. AI Integrations
        EmbeddingsGeneratorQueries = new EmbeddingsGenerator(this, ...);
        EmbeddingsGeneratorEtl = new EmbeddingsGenerator(this, ...);
        
        // 5. Features principais
        IndexStore = CreateIndexStore(serverStore);
        QueryRunner = new QueryRunner(this);
        EtlLoader = new EtlLoader(this, serverStore);
        QueueSinkLoader = new QueueSinkLoader(this, serverStore);
        SubscriptionStorage = CreateSubscriptionStorage(serverStore);
        OngoingTasks = new OngoingTasks.OngoingTasks(this);
        
        // 6. Infrastructure
        Metrics = new MetricCounters();
        MetricCacher = new DatabaseMetricCacher(this);
        TxMerger = new DocumentsTransactionOperationsMerger(this);
        ConfigurationStorage = new ConfigurationStorage(this);
        NotificationCenter = new DatabaseNotificationCenter(this);
        Operations = new DatabaseOperations(this);
        HugeDocuments = new HugeDocuments(NotificationCenter, ...);
        
        // 7. Cluster coordination
        RachisLogIndexNotifications = new DatabaseRaftIndexNotifications(...);
        ClusterWideTransactionIndexWaiter = new RaftIndexWaiter(DatabaseShutdown);
        CatastrophicFailureNotification = new CatastrophicFailureNotification(...);
        
        // 8. Background tasks
        CountersRepairTask = new CountersRepairTask(this, DatabaseShutdown);
        
        // 9. Proxy request executor
        _proxyRequestExecutor = CreateRequestExecutor();
        
        // 10. Certificate change handling
        _serverStore.Server.ServerCertificateChanged += OnCertificateChange;
    }
    catch (Exception)
    {
        Dispose(); // Cleanup em caso de erro
        throw;
    }
}
```

**?? Ordem Crítica:**
1. Loggers primeiro (para logging de erros)
2. Storage antes de features
3. TxMerger antes de qualquer operação de escrita
4. NotificationCenter cedo (para erros de inicialização)

### Fase 2: Initialize() - Startup dos Subsistemas

```csharp
public void Initialize(InitializeOptions options = InitializeOptions.None, 
    DateTime? wakeup = null)
{
    try
    {
        // 1. Security validation
        EnsureValidSecretKey();
        
        // 2. Directory permissions
        Configuration.CheckDirectoryPermissions();
        
        // 3. Compare Exchange Storage
        InitializeCompareExchangeStorage();
        
        // 4. Notification Center
        _addToInitLog(LogLevel.Debug, "Initializing NotificationCenter");
        NotificationCenter.Initialize();
        
        // 5. Documents Storage (CRITICAL!)
        _addToInitLog(LogLevel.Debug, "Initializing DocumentStorage");
        DocumentsStorage.Initialize((options & InitializeOptions.GenerateNewDatabaseId) 
            == InitializeOptions.GenerateNewDatabaseId);
        
        // 6. Transaction Merger
        _addToInitLog(LogLevel.Debug, "Starting Transaction Merger");
        TxMerger.Initialize(DocumentsStorage.ContextPool, IsEncrypted, Is32Bits);
        TxMerger.Start(); // ?? Inicia thread de merge
        
        // 7. Configuration Storage
        _addToInitLog(LogLevel.Debug, "Initializing ConfigurationStorage");
        ConfigurationStorage.Initialize();
        
        // 8. Cluster Transaction Error Notifier
        _clusterTransactionErrorNotifier.Initialize();
        
        if ((options & InitializeOptions.SkipLoadingDatabaseRecord) != 0)
            return; // Para cenários de teste
        
        _addToInitLog(LogLevel.Debug, "Loading Database");
        
        // 9. Metric Cacher
        MetricCacher.Initialize();
        
        // 10. Load Database Record do cluster
        DatabaseRecord record;
        long index;
        using (_serverStore.ContextPool.AllocateOperationContext(out var context))
        using (context.OpenReadTransaction())
        {
            record = _serverStore.Cluster.ReadDatabase(context, Name, out index);
        }
        
        if (record == null)
            DatabaseDoesNotExistException.Throw(Name);
        
        SetIds(record);
        OnDatabaseRecordChanged(record);
        SupportedFeatures = new SupportedFeature(record);
        
        // 11. Replication
        ReplicationLoader = CreateReplicationLoader();
        
        // 12. Periodic Backup
        PeriodicBackupRunner = new PeriodicBackupRunner(this, _serverStore, wakeup);
        
        // 13. Index Store (ASYNC - pode demorar!)
        _addToInitLog(LogLevel.Debug, "Initializing IndexStore (async)");
        _indexStoreTask = IndexStore.InitializeAsync(record, index, _addToInitLog);
        
        // 14. Replication Loader
        _addToInitLog(LogLevel.Debug, "Initializing Replication");
        ReplicationLoader?.Initialize(record, index);
        
        // 15. ETL
        _addToInitLog(LogLevel.Debug, "Initializing ETL");
        EtlLoader.Initialize(record);
        
        // 16. Queue Sinks
        _addToInitLog(LogLevel.Debug, "Initializing Queue Sinks");
        QueueSinkLoader.Initialize(record);
        
        // 17. Migrations (se necessário)
        InitializeAndStartDocumentsMigration();
        
        // 18. AGUARDAR IndexStore (BLOCKING!)
        try
        {
            _indexStoreTask.Wait();
        }
        finally
        {
            _indexStoreTask = null;
        }
        
        DatabaseShutdown.ThrowIfCancellationRequested();
        
        _addToInitLog(LogLevel.Debug, "Initializing SubscriptionStorage completed");
        
        // 19. Background cleaners
        TombstoneCleaner.Start();
        
        // 20. AI Features
        _addToInitLog(LogLevel.Debug, "Initializing Embeddings Generation");
        EmbeddingsGeneratorQueries.Start();
        EmbeddingsGeneratorEtl.Start();
        
        // 21. Storage Space Monitor
        _serverStore.StorageSpaceMonitor.Subscribe(this);
        
        // 22. Read last cluster transaction index
        using (DocumentsStorage.ContextPool.AllocateOperationContext(out var ctx))
        using (ctx.OpenReadTransaction())
        {
            var lastCompletedClusterTransactionIndex = 
                DocumentsStorage.ReadLastCompletedClusterTransactionIndex(
                    ctx.Transaction.InnerTransaction);
            ClusterWideTransactionIndexWaiter.SetAndNotifyListenersIfHigher(
                lastCompletedClusterTransactionIndex);
            
            // 23. Counter repair task (se necessário)
            var lastCounterFixed = DocumentsStorage.ReadLastFixedCounterKey(
                ctx.Transaction.InnerTransaction);
            if (lastCounterFixed != CountersRepairTask.Completed)
                _ = Task.Run(() => CountersRepairTask.Start(lastCounterFixed));
        }
        
        // 24. Notify features about state
        _ = Task.Run(async () =>
        {
            try
            {
                await DatabasesLandlord.NotifyFeaturesAboutStateChangeAsync(
                    record, index, _databaseStateChange, nameof(Initialize));
                RachisLogIndexNotifications.NotifyListenersAbout(index, e: null);
            }
            catch (Exception e)
            {
                RachisLogIndexNotifications.NotifyListenersAbout(index, e);
            }
        });
        
        // 25. Cluster Transaction Thread
        var clusterTransactionThreadName = ThreadNames.GetNameToUse(
            ThreadNames.ForClusterTransactions($"Cluster Transaction Thread {Name}", Name));
        
        _clusterTransactionsThread = PoolOfThreads.GlobalRavenThreadPool.LongRunning(
            x => ExecuteClusterTransaction(), 
            null, 
            clusterTransactionThreadName);
        
        // 26. Event subscriptions
        _serverStore.LicenseManager.LicenseChanged += 
            LoadTimeSeriesPolicyRunnerConfigurations;
        IoChanges.OnIoChange += CheckWriteRateAndNotifyIfNecessary;
    }
    catch (Exception)
    {
        Dispose();
        throw;
    }
}
```

**?? Observações Importantes:**

1. **Inicialização Assíncrona de Índices:**
   - `IndexStore.InitializeAsync()` retorna Task
   - Aguarda completion com `_indexStoreTask.Wait()`
   - Permite cancelamento via `DatabaseShutdown`

2. **Cluster Transaction Thread:**
   - Thread dedicada para processar cluster transactions
   - Long-running thread (não taskpool)
   - Priority: AboveNormal

3. **Error Handling:**
   - Qualquer exceção chama `Dispose()`
   - Garante cleanup mesmo em failures

---

## ?? Técnica de Performance #1: DatabaseUsage Pattern (RAII)

### Problema
Como prevenir unload de database enquanto está em uso?

### Solução: RAII Pattern com Struct

```csharp
public struct DatabaseUsage : IDisposable
{
    private readonly DocumentDatabase _parent;
    private readonly bool _skipUsagesCount;
    
    public DatabaseUsage(DocumentDatabase parent, bool skipUsagesCount)
    {
        _parent = parent;
        _skipUsagesCount = skipUsagesCount;
        
        if (_skipUsagesCount == false)
            Interlocked.Increment(ref _parent._usages);
        
        // Verifica shutdown DEPOIS de incrementar
        if (_parent.IsShutdownRequested())
        {
            Dispose();
            _parent.ThrowDatabaseShutdown();
        }
    }
    
    public void Dispose()
    {
        if (_skipUsagesCount)
            return;
        
        var currentUsagesCount = Interlocked.Decrement(ref _parent._usages);
        
        // Se shutdown + zero usages = liberar wait
        if (_parent._databaseShutdown.IsCancellationRequested && 
            currentUsagesCount == 0)
        {
            _parent._waitForUsagesOnDisposal.Set();
        }
    }
}

// Uso:
public DatabaseUsage DatabaseInUse(bool skipUsagesCount)
{
    return new DatabaseUsage(this, skipUsagesCount);
}
```

**No Route Handler:**
```csharp
public Task CreateDatabase(RequestHandlerContext context)
{
    var databaseName = context.RouteMatch.GetCapture();
    var databasesLandlord = context.RavenServer.ServerStore.DatabasesLandlord;
    var result = databasesLandlord.TryGetOrCreateDatabase(databaseName);
    
    // ... load database ...
    
    return context.Database.DatabaseShutdown.IsCancellationRequested == false
        ? Task.CompletedTask
        : UnlikelyWaitForDatabaseToUnload(...);
}

// No RequestRouter:
using (reqCtx.Database.DatabaseInUse(tryMatch.Value.SkipUsagesCount))
{
    await handler(reqCtx);
}
```

**?? Benefícios:**
- **Zero alocações:** Struct não aloca no heap
- **Thread-safe:** Interlocked operations
- **Automático:** using garante Dispose
- **Previne race conditions:** Incrementa ANTES de check

**?? Ganho:**
- Evita crashes por database unload durante operação
- Zero overhead de performance
- Proteção garantida via compiler

---

## ?? Técnica de Performance #2: Lazy Loading + Semaphore

### Problema
Evitar sobrecarga ao carregar muitas databases simultaneamente.

### Solução: Semaphore + Task Cache

```csharp
public sealed class DatabasesLandlord : IDisposable
{
    // Cache de tasks de loading
    public readonly ResourceCache<DocumentDatabase> DatabasesCache;
    
    // Limite de databases carregando simultaneamente
    internal SemaphoreSlim _databaseSemaphore;
    internal TimeSpan _concurrentDatabaseLoadTimeout;
    
    public DatabasesLandlord(ServerStore serverStore)
    {
        _databaseSemaphore = new SemaphoreSlim(
            _serverStore.Configuration.Databases.MaxConcurrentLoads);
        _concurrentDatabaseLoadTimeout = 
            _serverStore.Configuration.Databases.ConcurrentLoadTimeout.AsTimeSpan;
    }
    
    private Task<DocumentDatabase> CreateDatabaseUnderResourceSemaphore(
        StringSegment databaseName, RavenConfiguration config, DateTime? wakeup)
    {
        try
        {
            var task = new Task<DocumentDatabase>(() => 
                ActuallyCreateDatabase(databaseName, config, wakeup), 
                TaskCreationOptions.RunContinuationsAsynchronously);
            
            // GetOrAdd é atômico - apenas uma task é criada
            var database = DatabasesCache.GetOrAdd(databaseName, task);
            
            if (database == task)
            {
                // Esta thread venceu a race - inicia a task
                task.Start(); // Semaphore released no finally da task
                
                task.ContinueWith(t =>
                {
                    // Remove do cache de idle ao carregar com sucesso
                    _serverStore.IdleDatabases.TryRemove(databaseName.Value, out _);
                }, TaskContinuationOptions.OnlyOnRanToCompletion | 
                   TaskContinuationOptions.ExecuteSynchronously);
            }
            else
            {
                // Outra thread já está carregando - libera semaphore
                _databaseSemaphore.Release();
            }
            
            return database;
        }
        catch (Exception)
        {
            _databaseSemaphore.Release();
            throw;
        }
    }
    
    private async Task<DocumentDatabase> UnlikelyCreateDatabaseUnderContention(
        StringSegment databaseName, RavenConfiguration config, DateTime? wakeup)
    {
        var timeToWait = caller == Init 
            ? Timeout.InfiniteTimeSpan 
            : _concurrentDatabaseLoadTimeout;
        
        if (await _databaseSemaphore.WaitAsync(timeToWait) == false)
        {
            throw new DatabaseConcurrentLoadTimeoutException(
                "Too many databases loading concurrently, " +
                "timed out waiting for them to load.");
        }
        
        return await CreateDatabaseUnderResourceSemaphore(
            databaseName, config, wakeup);
    }
}
```

**?? Características:**

1. **Lazy Loading:**
   - Database só é carregado quando primeiro request chega
   - Cache em `DatabasesCache`
   - Task compartilhada entre múltiplos requests

2. **Semaphore Protection:**
   - Limite de databases carregando simultaneamente
   - Evita sobrecarga de I/O e memória
   - Timeout configurável

3. **Race Condition Handling:**
   - `GetOrAdd` é atômico
   - Apenas uma thread inicia a task
   - Outras threads esperam pela mesma task

**?? Configurações:**
```csharp
// appsettings.json
"Databases": {
    "MaxConcurrentLoads": 8,  // Default
    "ConcurrentLoadTimeout": "00:01:00"  // 1 minuto
}
```

---

## ?? Técnica de Performance #3: Idle Detection & Unload

### Problema
Databases não usados consomem memória desnecessariamente.

### Solução: Tracking de Último Acesso + Wakeup Timers

```csharp
public sealed class DatabasesLandlord
{
    public readonly ConcurrentDictionary<StringSegment, DateTime> LastRecentlyUsed;
    private readonly ConcurrentDictionary<string, Lazy<DatabaseWakeupTimer>> _wakeupTimers;
    
    public DateTime LastWork(DocumentDatabase resource)
    {
        if (ForTestingPurposes?.SkipIncreasingLastWorkTimeBasedOnDatabaseSize == true)
            return resource.LastAccessTime;
        
        // ?? Increase idle time baseado no tamanho do database
        // Adiciona 0.5 ms por KB = ~0.5s por MB
        var envs = resource.GetAllStoragesEnvironment();
        
        long dbSize = 0;
        var maxLastWork = resource.LastAccessTime;
        
        foreach (var env in envs)
        {
            dbSize += env.Environment.Stats().AllocatedDataFileSizeInBytes;
            
            if (env.Environment.LastWorkTime > maxLastWork)
                maxLastWork = env.Environment.LastWorkTime;
        }
        
        // Database maior = mais tempo idle antes de unload
        return maxLastWork.AddMilliseconds((dbSize / 1024L) * 0.5);
    }
    
    public bool UnloadDirectly(StringSegment databaseName, 
        IdleDatabaseActivity idleDatabaseActivity, string caller)
    {
        if (ShouldContinueDispose(databaseName.Value, idleDatabaseActivity) == false)
        {
            // Não unload se próximo wakeup é muito próximo
            return false;
        }
        
        if (DatabasesCache.TryGetValue(databaseName, out var databaseTask) == false)
            return false; // Já unloaded
        
        try
        {
            UnloadDatabaseInternal(databaseName.Value, caller);
            LastRecentlyUsed.TryRemove(databaseName, out _);
            
            if (idleDatabaseActivity?.DateTime != null)
            {
                // Agenda wakeup para próximo backup
                AddOrUpdateWakeupTimer(databaseName.Value, idleDatabaseActivity);
            }
            
            return true;
        }
        catch (Exception)
        {
            return false;
        }
    }
    
    private bool ShouldContinueDispose(string name, IdleDatabaseActivity activity)
    {
        if (activity?.DateTime == null)
            return true;
        
        // Não unload se wakeup é em menos de 5 minutos
        // (evita thrashing de load/unload)
        return activity.DueTime > TimeSpan.FromMinutes(5).TotalMilliseconds;
    }
}
```

**?? Heurísticas:**

1. **Tamanho do Database:**
   - Database grande = mais tempo idle
   - 1 MB = +0.5 segundos de idle time
   - Evita unload/reload de databases pesados

2. **Próximo Wakeup:**
   - Se backup agendado em <5min = não unload
   - Evita thrashing de I/O

3. **Last Work Time:**
   - Usa maior valor entre:
     - `resource.LastAccessTime`
     - `env.Environment.LastWorkTime` (mais preciso)

**?? Benefícios:**
- Reduz uso de memória
- Evita thrashing
- Mantém databases "quentes" quando necessário

---

## ?? Técnica de Performance #4: Catastrophic Failure Protection

### Problema
Corruption de dados pode causar crash loop infinito.

### Solução: Unload Automático com Limite

```csharp
public sealed class CatastrophicFailureHandler
{
    internal TimeSpan TimeToWaitBeforeUnloadingDatabase = TimeSpan.FromSeconds(2);
    internal int MaxDatabaseUnloads = 3;
    internal TimeSpan NoFailurePeriod = TimeSpan.FromMinutes(15);
    
    private readonly ConcurrentDictionary<Guid, FailureStats> _errorsPerEnvironment;
    
    public void Execute(string databaseName, Exception e, Guid environmentId, 
        string path, string stacktrace)
    {
        var stats = _errorsPerEnvironment.GetOrAdd(environmentId, 
            x => FailureStats.Create(MaxDatabaseUnloads));
        
        if (stats.WillUnloadDatabase == false)
        {
            // Já atingiu limite de 3 unloads
            if (DateTime.UtcNow - stats.LastUnloadTime > NoFailurePeriod)
            {
                // Resetar após 15 minutos sem falhas
                stats.NumberOfUnloads = 0;
                stats.LastUnloadTime = DateTime.MinValue;
            }
            else
            {
                return; // Não fazer mais nada
            }
        }
        
        stats.DatabaseUnloadTask = Task.Run(async () =>
        {
            // 1. Alerta antes de unload
            _serverStore.NotificationCenter.Add(AlertRaised.Create(
                databaseName,
                $"Critical error in '{databaseName}' database",
                $"Database will be unloaded due to error in: {path}",
                AlertReason.CatastrophicDatabaseFailure,
                NotificationSeverity.Error,
                details: new ExceptionDetails(e)));
            
            // 2. Esperar 2s para cliente receber resposta atual
            await Task.Delay(TimeToWaitBeforeUnloadingDatabase);
            
            // 3. Unload database
            stats.NumberOfUnloads++;
            stats.LastUnloadTime = DateTime.UtcNow;
            
            (await _databasesLandlord.UnloadAndLockDatabase(
                databaseName, "CatastrophicFailure"))?.Dispose();
        });
    }
}
```

**?? Proteções:**

1. **Limite de Unloads:**
   - Máximo 3 unloads automáticos
   - Evita crash loop infinito

2. **Reset Timer:**
   - Após 15min sem falhas, resetar contador
   - Permite recovery automático

3. **Delay Before Unload:**
   - 2 segundos antes de unload
   - Cliente recebe resposta de erro
   - Evita broken pipe

**?? Cenários:**
- Voron corruption
- Disk failure
- Memory corruption
- Index corruption

---

## ?? Padrões de Design

### 1. **RAII Pattern (DatabaseUsage)**
```csharp
public struct DatabaseUsage : IDisposable
{
    public DatabaseUsage(DocumentDatabase parent, bool skipUsagesCount)
    {
        // Acquire resource
        if (_skipUsagesCount == false)
            Interlocked.Increment(ref _parent._usages);
    }
    
    public void Dispose()
    {
        // Release resource
        var count = Interlocked.Decrement(ref _parent._usages);
        if (count == 0) _parent._waitForUsagesOnDisposal.Set();
    }
}
```

**Por que?**
- Garantia de cleanup via `using`
- Thread-safe resource tracking
- Zero alocações (struct)

### 2. **Lazy Initialization**
```csharp
public DatabaseSearchResult TryGetOrCreateDatabase(StringSegment databaseName)
{
    if (TryGetResourceStore(databaseName, out var databaseTask))
        return new DatabaseSearchResult(Status.Database, databaseTask, null);
    
    // Lazy loading sob demanda
    return new DatabaseSearchResult(Status.Database, 
        TryGetOrCreateResourceStore(databaseName), null);
}
```

**Por que?**
- Reduz uso de memória
- Startup mais rápido
- Load sob demanda

### 3. **Dispose Once Pattern**
```csharp
private readonly DisposeOnce<SingleAttempt> _disposeOnce;

public void Dispose()
{
    _disposeOnce.Dispose();
}

private unsafe void DisposeInternal()
{
    // Dispose logic aqui
    // Chamado apenas uma vez
}
```

**Por que?**
- Thread-safe
- Idempotent
- Previne double-dispose

### 4. **Circuit Breaker (Catastrophic Failure)**
```csharp
if (stats.NumberOfUnloads >= MaxDatabaseUnloads)
{
    // Circuit open - não tenta mais
    if (DateTime.UtcNow - stats.LastUnloadTime < NoFailurePeriod)
        return;
    
    // Reset após período sem falhas
    stats.NumberOfUnloads = 0;
}
```

**Por que?**
- Evita crash loops
- Auto-recovery
- Fail-safe

### 5. **Coordinator Pattern**
```csharp
public class DocumentDatabase
{
    // Coordena 20+ subsistemas
    public DocumentsStorage DocumentsStorage { get; }
    public IndexStore IndexStore { get; }
    public ReplicationLoader ReplicationLoader { get; }
    public EtlLoader EtlLoader { get; }
    // ... 16+ outros subsistemas
    
    public void Initialize()
    {
        // Coordena inicialização em ordem específica
        DocumentsStorage.Initialize();
        TxMerger.Initialize(...);
        TxMerger.Start();
        IndexStore.InitializeAsync(...);
        // ... coordenação precisa
    }
}
```

**Por que?**
- Centraliza orquestração
- Garante ordem correta
- Facilita debugging

---

## ?? Tratamento de Erros

### 1. **Constructor Failure**
```csharp
public DocumentDatabase(...)
{
    try
    {
        // Criar todos os subsistemas
    }
    catch (Exception)
    {
        Dispose(); // ?? Cleanup automático
        throw;
    }
}
```

**Estratégia:** Cleanup automático em caso de erro

### 2. **Initialize Failure**
```csharp
public void Initialize(...)
{
    try
    {
        // Inicializar subsistemas
    }
    catch (Exception)
    {
        Dispose();
        throw;
    }
}
```

**Estratégia:** Mesmo que constructor

### 3. **Load Timeout**
```csharp
private async Task UnlikelyCreateDatabaseUnderContention(...)
{
    if (await _databaseSemaphore.WaitAsync(timeToWait) == false)
    {
        throw new DatabaseConcurrentLoadTimeoutException(
            "Too many databases loading concurrently, " +
            "timed out waiting for them to load.");
    }
    // ...
}
```

**Estratégia:** Timeout configurável + exceção específica

### 4. **Graceful Shutdown**
```csharp
private void DisposeInternal()
{
    _databaseShutdown.Cancel();
    
    // 1. Drain all requests (até 60s)
    var sp = Stopwatch.StartNew();
    while (sp.ElapsedMilliseconds < 60 * 1000)
    {
        if (Interlocked.Read(ref _usages) == 0)
            break;
        
        if (_waitForUsagesOnDisposal.Wait(1000))
            _waitForUsagesOnDisposal.Reset();
    }
    
    // 2. Dispose subsistemas em ordem reversa
    var exceptionAggregator = new ExceptionAggregator(_logger, ...);
    
    exceptionAggregator.Execute(() => TxMerger?.Dispose());
    exceptionAggregator.Execute(() => IndexStore?.Dispose());
    // ... todos os subsistemas
    
    exceptionAggregator.ThrowIfNeeded();
}
```

**Estratégia:** Aguardar drain + dispose ordenado + aggregate exceptions

---

## ?? Métricas & Observabilidade

### 1. **Initialization Log**
```csharp
public ConcurrentDictionary<string, ConcurrentQueue<string>> InitLog =
    new ConcurrentDictionary<string, ConcurrentQueue<string>>();

private void AddToInitLog(LogLevel logMode, string txt)
{
    string msg = $"[Load Database] {DateTime.UtcNow} :: Database '{databaseName}' : {txt}";
    
    if (InitLog.TryGetValue(databaseName.Value, out var q))
        q.Enqueue(msg);
    
    // Log também vai para arquivo
    _logger.Info(msg);
}
```

**Visível em:**
- Studio UI durante loading
- Server logs
- Debug endpoint `/debug/databases/init-log`

### 2. **Database Info Cache**
```csharp
public DynamicJsonValue GenerateOfflineDatabaseInfo()
{
    var sizeOnDisk = GetSizeOnDisk();
    var indexingErrors = IndexStore.GetIndexes().Sum(index => index.GetErrorCount());
    var alertCount = NotificationCenter.GetAlertCount();
    var performanceHints = NotificationCenter.GetPerformanceHintCount();
    var backupInfo = PeriodicBackupRunner?.GetBackupInfo();
    var mountPointsUsage = GetMountPointsUsage(includeTempBuffers: false);
    var documentsCount = DocumentsStorage.GetNumberOfDocuments();
    var indexesCount = IndexStore.GetIndexes().Count();
    
    return new DynamicJsonValue
    {
        [nameof(ExtendedDatabaseInfo.TotalSize)] = sizeOnDisk.Data,
        [nameof(ExtendedDatabaseInfo.IndexingErrors)] = indexingErrors,
        [nameof(ExtendedDatabaseInfo.Alerts)] = alertCount,
        [nameof(ExtendedDatabaseInfo.DocumentsCount)] = documentsCount,
        [nameof(ExtendedDatabaseInfo.IndexesCount)] = indexesCount,
        // ... mais informações
    };
}
```

**Usado para:**
- Studio UI quando database está unloaded
- Health checks
- Monitoring

### 3. **Usage Tracking**
```csharp
private long _usages;

public DatabaseUsage DatabaseInUse(bool skipUsagesCount)
{
    return new DatabaseUsage(this, skipUsagesCount);
}
```

**Métricas:**
- `_usages`: Número de operações em andamento
- Usado para determinar quando é seguro unload
- Visível em debug endpoints

---

## ?? Integração com Outros Módulos

### Consome

| Módulo | Uso |
|--------|-----|
| **Módulo 1 (HTTP Pipeline)** | RouteInformation chama `CreateDatabase()` |
| **Módulo 2 (Transaction Merger)** | TxMerger é criado e inicializado aqui |
| **Voron** | DocumentsStorage, ConfigurationStorage |
| **ServerStore** | Lê DatabaseRecord do cluster |
| **NotificationCenter** | Alertas de loading/errors |

### É Consumido Por

| Módulo | Uso |
|--------|-----|
| **Módulo 1 (HTTP Pipeline)** | Handlers usam `DatabaseInUse()` |
| **Módulo 4 (Document Storage)** | CRUD operations |
| **Módulo 5 (Indexing)** | Index processing |
| **Módulo 7 (Replication)** | Incoming/Outgoing replication |
| **Background Tasks** | Backup, Expiration, etc. |

---

## ?? Lições Aprendidas

### ? DO's (Faça)

```
? Use RAII pattern para resource tracking
? Implemente graceful shutdown com timeout
? Agregue exceções durante dispose
? Use semaphore para limitar cargas concorrentes
? Log detalhado durante inicialização
? Drain requests antes de unload
? Cache database info para quando unloaded
? Implemente circuit breaker para catastrophic failures
? Use CancellationToken linkado ao server shutdown
? Ordem de inicialização é crítica - documente!
```

### ? DON'Ts (Não Faça)

```
? Não bloqueie constructor - use Initialize()
? Não ignore exceções durante dispose
? Não unload database com requests ativas
? Não use Task.Run() sem tratar exceções
? Não esqueça Dispose() em caso de erro no constructor
? Não inicie threads antes de Initialize() completo
? Não confie em ordem aleatória de subsistemas
? Não faça lazy init de componentes críticos (storage)
```

### ?? Anti-Patterns Evitados

**1. Inicialização Parcial:**
```csharp
// ? MAU - deixa objeto em estado inválido
public DocumentDatabase(...)
{
    DocumentsStorage = new DocumentsStorage(this);
    // Exception aqui deixa DocumentsStorage criado mas não inicializado
    TxMerger = new DocumentsTransactionOperationsMerger(this);
}

// ? BOM - cleanup automático
public DocumentDatabase(...)
{
    try
    {
        DocumentsStorage = new DocumentsStorage(this);
        TxMerger = new DocumentsTransactionOperationsMerger(this);
    }
    catch
    {
        Dispose(); // Limpa tudo
        throw;
    }
}
```

**2. Double Dispose:**
```csharp
// ? MAU - pode chamar Dispose() múltiplas vezes
public void Dispose()
{
    TxMerger?.Dispose();
    IndexStore?.Dispose();
}

// ? BOM - garante single execution
private readonly DisposeOnce<SingleAttempt> _disposeOnce;

public void Dispose()
{
    _disposeOnce.Dispose();
}
```

---

## ?? Referências

### Código-Fonte Principal
- `src/Raven.Server/Documents/DocumentDatabase.cs` (4500+ linhas!)
- `src/Raven.Server/Documents/DatabasesLandlord.cs` (2500+ linhas)
- `src/Raven.Server/Documents/CatastrophicFailureHandler.cs`
- `src/Raven.Server/Documents/InitializeOptions.cs`
- `src/Raven.Server/Documents/DatabaseLoggersContext.cs`

### Conceitos Importantes
- **RAII Pattern**: Resource Acquisition Is Initialization
- **Lazy Loading**: Carregar sob demanda
- **Circuit Breaker**: Proteção contra failures
- **Graceful Shutdown**: Drain + dispose ordenado
- **Semaphore**: Controle de concorrência

### Configurações Importantes
```json
{
    "Databases": {
        "MaxConcurrentLoads": 8,
        "ConcurrentLoadTimeout": "00:01:00"
    },
    "Server": {
        "MaxTimeForTaskToWaitForDatabaseToLoad": "00:01:00"
    }
}
```

### Ordem de Inicialização (Crítica!)

```
1. Loggers
2. Configuration
3. Shutdown coordination (_databaseShutdown)
4. File locking
5. Basic subsystems (Smuggler, QueryMetadataCache, IoChanges, Changes)
6. TombstoneCleaner
7. DocumentsStorage ?
8. CompareExchangeStorage
9. AI Integrations
10. IndexStore
11. QueryRunner
12. EtlLoader, QueueSinkLoader
13. SubscriptionStorage
14. OngoingTasks
15. Infrastructure (Metrics, MetricCacher, NotificationCenter, Operations)
16. TxMerger ?
17. ConfigurationStorage
18. Cluster coordination (RachisLogIndexNotifications, ClusterWideTransactionIndexWaiter)
19. CatastrophicFailureNotification
20. Background tasks (CountersRepairTask)
21. Proxy request executor
```

---

## ?? Resumo Executivo

O **DocumentDatabase Lifecycle** é o módulo responsável por:

1. **Lazy Loading:** Databases carregados sob demanda com semaphore protection
2. **Coordenação:** Inicialização ordenada de 20+ subsistemas
3. **RAII Protection:** DatabaseUsage pattern previne unload durante uso
4. **Idle Management:** Detecção automática + wakeup timers para backups
5. **Graceful Shutdown:** Drain requests + dispose ordenado
6. **Catastrophic Protection:** Circuit breaker automático contra corruption
7. **Observabilidade:** Init logs, database info cache, métricas

**Complexidade:** ???? ALTA  
**LOC Analisadas:** ~7,000+ linhas  
**Tempo de Análise:** ~3-4 horas

**Próximo Módulo Recomendado:** Módulo 4 (Document Storage & CRUD) para entender como documentos são armazenados.

---

**? Módulo 3 Completo!**
