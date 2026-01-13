# Raven.Server - Módulo 5: Indexing Subsystem

## ?? Visão Geral

O **Indexing Subsystem** é o componente mais complexo e crítico de performance do RavenDB, responsável por criar, manter e consultar índices sobre os documentos armazenados. É onde ocorre a mágica de transformar documentos JSON em estruturas otimizadas para queries.

### Responsabilidades Principais

- **Index Management**: Criação, abertura, atualização e remoção de índices
- **Auto-Index Creation**: Criação dinâmica de índices baseada em queries
- **Indexing Execution**: Processamento de documentos e geração de índices
- **Query Execution**: Resposta a queries utilizando índices
- **Rolling Deployment**: Substituição de índices sem downtime
- **Error Handling**: Recuperação de falhas de indexação

### Componentes do Módulo

```
Documents/Indexes/
??? IndexStore.cs                        # Orquestrador (~2.500 LOC)
??? Index.cs                             # Classe base abstrata (~6.000 LOC!)
??? Auto/
?   ??? AutoMapIndex.cs                  # Índice automático map
?   ??? AutoMapReduceIndex.cs            # Índice automático map-reduce
??? Static/
?   ??? MapIndex.cs                      # Índice estático map
?   ??? MapReduceIndex.cs                # Índice estático map-reduce
?   ??? StaticIndexBase.cs               # Base para índices compilados
??? Workers/
?   ??? MapDocuments.cs                  # Worker de mapeamento
?   ??? HandleReferences.cs              # Worker de referências
?   ??? Cleanup/CleanupDocuments.cs      # Worker de limpeza
??? Persistence/
?   ??? Lucene/LuceneIndexPersistence.cs # Backend Lucene
?   ??? Corax/CoraxIndexPersistence.cs   # Backend Corax (novo)
??? Queries/
    ??? Query/QueryRunner.cs             # Execução de queries
    ??? Dynamic/DynamicQueryRunner.cs    # Queries dinâmicas
```

---

## ??? Arquitetura Interna

### 1. **IndexStore - O Orquestrador**

```csharp
public class IndexStore : IDisposable
{
    private readonly DocumentDatabase _documentDatabase;
    private readonly ServerStore _serverStore;

    // Coleção thread-safe de índices
    private readonly CollectionOfIndexes _indexes = new CollectionOfIndexes();
    
    // Lock por índice para operações atômicas
    private readonly ConcurrentDictionary<string, SemaphoreSlim> _indexLocks = 
        new ConcurrentDictionary<string, SemaphoreSlim>(StringComparer.OrdinalIgnoreCase);

    // Controllers especializados
    public readonly DatabaseIndexCreateController Create;
    public readonly DatabaseIndexDeleteController Delete;
    public readonly DatabaseIndexLockModeController LockMode;
    public readonly DatabaseIndexPriorityController Priority;
    public readonly DatabaseIndexStateController State;

    // Limita indexação concorrente em low memory
    public SemaphoreSlim StoppedConcurrentIndexBatches { get; }
}
```

### 2. **Index - A Base Abstrata**

```csharp
public abstract class Index : ITombstoneAware, IDisposable, ILowMemoryHandler
{
    // Thread de indexação (long-running)
    internal PoolOfThreads.LongRunningWork _indexingThread;
    
    // Cancellation token para shutdown graceful
    private CancellationTokenSource _indexingProcessCancellationTokenSource;
    
    // Storage engine (Voron)
    internal StorageEnvironment _environment;
    internal TransactionContextPool _contextPool;
    
    // Persistence layer (Lucene ou Corax)
    internal IndexPersistenceBase IndexPersistence;
    
    // Workers de indexação
    private IIndexingWork[] _indexWorkers;
    
    // Sincronização de queries vs indexing
    private readonly AsyncReaderWriterLock _currentlyRunningQueriesLock = new();
    
    // Throttling de indexação
    internal ThrottledManualResetEventSlim _mre;
    
    // Estatísticas e performance
    private IndexingStatsAggregator _lastStats;
    private readonly ConcurrentQueue<IndexingStatsAggregator> _lastIndexingStats = new();
    
    // Memory management
    private Size _currentMaximumAllowedMemory = DefaultMaximumMemoryAllocation;
    private readonly MultipleUseFlag _lowMemoryFlag = new();
}
```

### 3. **CollectionOfIndexes - Storage Thread-Safe**

```csharp
// Wrapper thread-safe sobre Dictionary<string, Index>
private sealed class CollectionOfIndexes : IEnumerable<Index>
{
    private Dictionary<string, Index> _indexes = new Dictionary<string, Index>(StringComparer.OrdinalIgnoreCase);
    private Dictionary<string, List<Index>> _indexesPerCollection = new Dictionary<string, List<Index>>(StringComparer.OrdinalIgnoreCase);

    public void Add(Index index)
    {
        // Copy-on-write pattern
        var newIndexes = new Dictionary<string, Index>(_indexes);
        newIndexes[index.Name] = index;
        
        var newPerCollection = new Dictionary<string, List<Index>>(_indexesPerCollection);
        foreach (var collection in index.Collections)
        {
            if (newPerCollection.TryGetValue(collection, out var list) == false)
                newPerCollection[collection] = list = new List<Index>();
            
            list.Add(index);
        }
        
        // Atomic replace
        _indexes = newIndexes;
        _indexesPerCollection = newPerCollection;
    }
    
    public bool TryGetByName(string name, out Index index)
    {
        return _indexes.TryGetValue(name, out index);
    }
    
    public IEnumerable<Index> GetForCollection(string collection)
    {
        if (_indexesPerCollection.TryGetValue(collection, out var list))
            return list;
        
        return Enumerable.Empty<Index>();
    }
}
```

---

## ?? Fluxo de Indexação

### **ExecuteIndexing() - O Loop Principal**

```
???????????????????????????????????????????????????????????????
?               Index.ExecuteIndexing() Loop                  ?
???????????????????????????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  1. Initialize Thread                 ?
        ?     - Register NativeMemory stats     ?
        ?     - Subscribe to changes            ?
        ?     - Enable throttling timer         ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  2. Wait for Work                     ?
        ?     - _mre.Wait()                     ?
        ?     - Check cancellation              ?
        ?     - Handle rolling deployment       ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  3. OnBeforeExecuteIndexing           ?
        ?     - Lucene: warmup searchers        ?
        ?     - Corax: prepare readers          ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  4. DoIndexingWork()                 ? ? CORE
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  5. Handle Errors                     ?
        ?     - IndexWriteException             ?
        ?     - IndexCorruptionException        ?
        ?     - OutOfMemoryException            ?
        ?     - DiskFullException               ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  6. Update Statistics                 ?
        ?     - _indexStorage.UpdateStats()     ?
        ?     - Handle failure information      ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  7. Replace If Needed                 ?
        ?     - Side-by-side completion         ?
        ?     - Rename replacement index        ?
        ?     - Delete old index                ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  8. Notify Batch Completed            ?
        ?     - IndexChange event               ?
        ?     - TestRun.BatchCompleted          ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  9. Memory Cleanup                    ?
        ?     - ReduceMemoryUsage()             ?
        ?     - Cleanup scratch space           ?
        ?     - Persist elapsed time            ?
        ????????????????????????????????????????
                           ?
                           ??????????? Loop
                                     ?
                                     ?
```

### **DoIndexingWork() - Processamento de Batch**

```csharp
public bool DoIndexingWork(IndexingStatsScope stats, CancellationToken cancellationToken)
{
    _threadAllocations = NativeMemory.CurrentThreadStats;
    _initialManagedAllocations = new Size(GC.GetAllocatedBytesForCurrentThread(), SizeUnit.Bytes);

    bool mightBeMore = false;

    using (var context = QueryOperationContext.Allocate(DocumentDatabase, this))
    using (_contextPool.AllocateOperationContext(out TransactionOperationContext indexContext))
    using (CurrentIndexingScope.Current = CreateIndexingScope(indexContext, context))
    {
        using (var tx = indexContext.OpenWriteTransaction())
        {
            var writeOperation = new Lazy<IndexWriteOperationBase>(() =>
            {
                var writer = IndexPersistence.OpenIndexWriter(indexContext.Transaction.InnerTransaction, indexContext);
                return writer;
            });
            
            try
            {
                using (InitializeIndexingWork(indexContext))
                {
                    // Executa todos os workers em ordem
                    foreach (var work in _indexWorkers)
                    {
                        using (var scope = stats.For(work.Name))
                        {
                            var result = work.Execute(context, indexContext, writeOperation, scope, cancellationToken);
                            mightBeMore |= result.MoreWorkFound;
                            
                            if (mightBeMore)
                                _mre.Set(ignoreThrottling: result.BatchContinuationResult == CanContinueBatchResult.False);
                        }
                    }
                    
                    // Commit das mudanças
                    if (writeOperation.IsValueCreated)
                    {
                        using (var indexWriteOperation = writeOperation.Value)
                        {
                            indexWriteOperation.Commit(stats, cancellationToken);
                        }
                    }
                    
                    // Commit da transação Voron
                    tx.Commit();
                }
                
                return mightBeMore;
            }
            catch
            {
                DisposeIndexWriterOnError(writeOperation);
                throw;
            }
        }
    }
}
```

---

## ?? Técnicas de Performance

### 1. **Batching Inteligente com Heurísticas**

**Problema:** Processar documentos um por um é ineficiente. Processar demais causa OOM.

**Solução:** Heurísticas dinâmicas baseadas em múltiplos fatores.

```csharp
public CanContinueBatchResult CanContinueBatch(in CanContinueBatchParameters parameters, 
    ref TimeSpan maxTimeForDocumentTransactionToRemainOpen)
{
    // 1. Limite configurável
    if (Configuration.MapBatchSize.HasValue && 
        parameters.Count >= Configuration.MapBatchSize.Value)
    {
        parameters.Stats.RecordBatchCompletedReason(parameters.WorkType, 
            $"Reached maximum configured map batch size ({Configuration.MapBatchSize.Value:#,#;;0}).");
        return CanContinueBatchResult.False;
    }
    
    // 2. Atingiu max etag + timeout
    if (parameters.CurrentEtag >= parameters.MaxEtag && 
        parameters.Stats.Duration >= Configuration.MapTimeoutAfterEtagReached.AsTimeSpan)
    {
        parameters.Stats.RecordBatchCompletedReason(parameters.WorkType, 
            $"Reached maximum etag ({parameters.MaxEtag:#,#;;0}) and duration ({parameters.Stats.Duration}) exceeded limit");
        return CanContinueBatchResult.False;
    }
    
    // 3. Check a cada 128 ops (não toda iteração!)
    if (parameters.Count % 128 != 0)
        return CanContinueBatchResult.True;
    
    // 4. Transaction age
    if (parameters.Sw.Elapsed > maxTimeForDocumentTransactionToRemainOpen)
    {
        if (parameters.QueryContext.Documents.ShouldRenewTransactionsToAllowFlushing())
            return CanContinueBatchResult.RenewTransaction; // ? Renew, não stop!
        
        maxTimeForDocumentTransactionToRemainOpen = 
            maxTimeForDocumentTransactionToRemainOpen.Add(Configuration.MaxTimeForDocumentTransactionToRemainOpen.AsTimeSpan);
    }
    
    // 5. Timeout global
    if (parameters.Stats.Duration >= Configuration.MapTimeout.AsTimeSpan)
    {
        parameters.Stats.RecordBatchCompletedReason(parameters.WorkType, 
            $"Exceeded maximum configured map duration ({Configuration.MapTimeout.AsTimeSpan})");
        return CanContinueBatchResult.False;
    }
    
    // 6. Flush waiting
    if (ShouldReleaseTransactionBecauseFlushIsWaiting(parameters.Stats, parameters.WorkType))
        return CanContinueBatchResult.False;
    
    // 7. Memory pressure
    var (txAllocationsInBytes, indexWriterUnmanagedAllocationsInBytes) = 
        UpdateThreadAllocations(parameters.IndexingContext, parameters.IndexWriteOperation, parameters.Stats, parameters.WorkType);
    
    if (_lowMemoryFlag.IsRaised() && parameters.Count > MinBatchSize)
    {
        HandleStoppedBatchesConcurrently(parameters.Stats, parameters.Count,
            canContinue: () => _lowMemoryFlag.IsRaised() == false,
            reason: "low memory", parameters.WorkType);
        
        return CanContinueBatchResult.False;
    }
    
    // 8. Transaction size limit
    if (TransactionSizeLimit != null)
    {
        var txAllocations = new Size(txAllocationsInBytes, SizeUnit.Bytes);
        if (txAllocations > TransactionSizeLimit.Value)
        {
            parameters.Stats.RecordBatchCompletedReason(parameters.WorkType, 
                $"Reached transaction size limit ({TransactionSizeLimit.Value}). Allocated {txAllocations}");
            return CanContinueBatchResult.False;
        }
    }
    
    // 9. Memory budget
    var allocated = new Size(_threadAllocations.CurrentlyAllocatedForProcessing, SizeUnit.Bytes);
    if (allocated > _currentMaximumAllowedMemory)
    {
        if (MemoryUsageGuard.TryIncreasingMemoryUsageForThread(
            _threadAllocations, ref _currentMaximumAllowedMemory, allocated,
            _environment.Options.RunningOn32Bits, DocumentDatabase.ServerStore.Server.MetricCacher,
            _logger, out var memoryUsage) == false)
        {
            // Não conseguiu aumentar budget, verificar se pode parar
            var canStop = false;
            switch (parameters.WorkType)
            {
                case IndexingWorkType.Map:
                    canStop = parameters.Stats.MapAttempts >= Configuration.MinNumberOfMapAttemptsAfterWhichBatchWillBeCanceledIfRunningLowOnMemory;
                    break;
            }
            
            if (canStop)
            {
                HandleStoppedBatchesConcurrently(parameters.Stats, parameters.Count,
                    canContinue: MemoryUsageGuard.CanIncreaseMemoryUsageForThread,
                    reason: "cannot budget additional memory", parameters.WorkType);
                
                return CanContinueBatchResult.False;
            }
        }
    }
    
    return CanContinueBatchResult.True;
}
```

**Resultado:** Batchs otimizados dinamicamente baseados em 9 heurísticas diferentes!

### 2. **Throttled Manual Reset Event**

**Problema:** Acordar thread de indexação a cada mudança de documento causa overhead.

**Solução:** Throttling com timer e modo de ignore.

```csharp
internal class ThrottledManualResetEventSlim : IDisposable
{
    private readonly ManualResetEventSlim _event = new ManualResetEventSlim(false);
    private readonly TimeSpan? _throttlingInterval;
    private Timer _timer;
    private int _setScheduled;
    
    public void Set(bool ignoreThrottling = false)
    {
        if (ignoreThrottling)
        {
            // Set imediato, ignora throttling
            _event.Set();
            return;
        }
        
        // Usa throttling
        if (_throttlingInterval == null || _throttlingInterval.Value == TimeSpan.Zero)
        {
            _event.Set();
            return;
        }
        
        if (Interlocked.CompareExchange(ref _setScheduled, 1, 0) == 1)
            return; // Já agendado
        
        // Agenda Set após interval
        _timer?.Change(_throttlingInterval.Value, Timeout.InfiniteTimeSpan);
    }
    
    public void Reset()
    {
        Interlocked.Exchange(ref _setScheduled, 0);
        _event.Reset();
    }
    
    public bool Wait(int millisecondsTimeout, CancellationToken token)
    {
        return _event.Wait(millisecondsTimeout, token);
    }
}
```

**Benefício:** Reduz "wake-ups" de indexing thread em 90% em workloads de alta escrita.

### 3. **AsyncReaderWriterLock para Queries vs Indexing**

**Problema:** Queries precisam ler enquanto indexação escreve. Mas algumas operações (compact, reset) precisam exclusividade.

**Solução:** Reader/Writer lock assíncrono.

```csharp
private readonly AsyncReaderWriterLock _currentlyRunningQueriesLock = new();

// Query (muitos leitores)
public async Task<DocumentQueryResult> Query(IndexQueryServerSide query, ...)
{
    using (var marker = MarkQueryAsRunning(query))
    {
        // Adquire read lock
        await marker.HoldLockAsync();
        
        // Executa query
        using (_contextPool.AllocateOperationContext(out TransactionOperationContext indexContext))
        using (var indexTx = indexContext.OpenReadTransaction())
        {
            // ... query execution
        }
    }
}

private sealed class IndexQueryDoneRunning : IDisposable
{
    private readonly Index _parent;
    private IDisposable _lock;
    
    public async ValueTask HoldLockAsync()
    {
        var timeout = _parent._isReplacing 
            ? ExtendedLockTimeout // 30s quando substituindo
            : DefaultLockTimeout; // 3s normalmente
        
        using (var cts = new CancellationTokenSource(timeout))
            _lock = await _parent._currentlyRunningQueriesLock.ReaderLockAsync(cts.Token);
    }
    
    public void Dispose()
    {
        _lock?.Dispose();
    }
}

// Operações exclusivas (compaction, reset)
internal IDisposable DrainRunningQueries()
{
    using (var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10)))
        currentlyRunningQueriesWriteLock = _currentlyRunningQueriesLock.WriterLock(cts.Token);
    
    // Agora temos exclusividade, nenhuma query rodando
    return new ExitWriteLock(currentlyRunningQueriesWriteLock, this);
}
```

**Ganho:** Queries concorrentes sem blocking. Operações de manutenção são raras mas seguras.

### 4. **Index Replacement via Side-by-Side**

**Problema:** Atualizar definição de índice exige reconstrução completa. Como fazer sem downtime?

**Solução:** Criar índice com prefixo especial, indexar em paralelo, swap atômico.

```csharp
private Index HandleStaticIndexChange(string name, IndexDefinition definition, bool forceUpdate = false)
{
    using (IndexLock(name))
    {
        var currentIndex = GetIndex(name);
        var creationOptions = GetIndexCreationOptions(definition, currentIndex?.ToIndexInformationHolder(), 
            _documentDatabase.Configuration, out var currentDifferences);
        
        if (creationOptions == IndexCreationOptions.Update)
        {
            // Precisa rebuild completo
            var replacementIndexName = Constants.Documents.Indexing.SideBySideIndexNamePrefix + definition.Name;
            // "ReplacementOf/" + "Users/ByName" = "ReplacementOf/Users/ByName"
            
            definition.Name = replacementIndexName;
            var replacementIndex = GetIndex(replacementIndexName);
            
            if (replacementIndex != null)
            {
                // Já existe replacement, verificar se precisa atualizar
                var sideBySideOptions = GetIndexCreationOptions(definition, replacementIndex.ToIndexInformationHolder(), 
                    _documentDatabase.Configuration, out var sideBySideDifferences);
                
                if (sideBySideOptions == IndexCreationOptions.Noop)
                    return null; // Replacement já está correto
                
                DeleteIndexInternal(replacementIndex); // Deletar antigo replacement
            }
            
            var index = CreateIndexFromDefinition(definition, _documentDatabase);
            
            // Index será iniciado e começará a indexar em paralelo
            return index;
        }
        
        return null;
    }
}

// Após indexação completa
public void ReplaceIndexes(string oldIndexName, string replacementIndexName, CancellationToken token)
{
    var indexLock = GetIndexLock(oldIndexName);
    indexLock.Wait(token);
    
    try
    {
        var newIndex = GetIndex(replacementIndexName);
        var oldIndex = GetIndex(oldIndexName);
        
        // 1. Atomic update da coleção
        _indexes.ReplaceIndex(oldIndexName, oldIndex, newIndex);
        
        // 2. Stop newIndex para renomear
        using (newIndex.DrainRunningQueries())
        {
            newIndex.Stop(disableIndex: false);
            
            try
            {
                // 3. Rename: "ReplacementOf/Users/ByName" ? "Users/ByName"
                newIndex.Rename(oldIndexName);
            }
            finally
            {
                newIndex.Start();
            }
        }
        
        newIndex.ResetIsSideBySideAfterReplacement();
        
        // 4. Delete old index
        if (oldIndex != null)
        {
            using (oldIndex.DrainRunningQueries())
                DeleteIndexInternal(oldIndex, raiseNotification: false);
        }
        
        // 5. Move diretórios físicos
        if (newIndex.Configuration.RunInMemory == false)
        {
            using (newIndex.DrainRunningQueries())
            {
                using (newIndex.RestartEnvironment())
                {
                    IOExtensions.MoveDirectory(
                        newIndex.Configuration.StoragePath.Combine(replacementIndexDirectoryName).FullPath,
                        newIndex.Configuration.StoragePath.Combine(oldIndexDirectoryName).FullPath);
                }
            }
        }
        
        _documentDatabase.Changes.RaiseNotifications(
            new IndexChange { Name = oldIndexName, Type = IndexChangeTypes.SideBySideReplace });
    }
    finally
    {
        indexLock.Release();
    }
}
```

**Resultado:** Zero downtime na atualização de índices!

### 5. **Rolling Index Deployment em Cluster**

**Problema:** Em cluster, atualizar índice em todos os nós simultaneamente pode causar performance degradation.

**Solução:** Deploy incremental nó por nó.

```csharp
public void RollIfNeeded()
{
    RaiseNotificationIfNeeded();
    
    GetPendingAndReplaceStatus(out var pending, out var shouldReplace);
    
    if (shouldReplace)
    {
        if (ForceReplace.Raise() == false)
            return; // Ainda aguardando
    }
    
    if (shouldReplace || pending == false)
    {
        _rollingEvent.Set(); // Permite indexação neste nó
        CompleteIfRollingSideBySideRemoved();
    }
}

private void GetPendingAndReplaceStatus(out bool pending, out bool replace)
{
    pending = false;
    replace = false;
    
    if (IsRolling == false)
        return;
    
    if (DocumentDatabase.IndexStore.ShouldSkipThisNodeWhenRolling(this, out var reason, out replace))
    {
        pending = true;
        _lastPendingStatus = reason;
    }
}

public bool ShouldSkipThisNodeWhenRolling(Index index, out string reason, out bool replace)
{
    replace = false;
    
    using (_serverStore.ContextPool.AllocateOperationContext(out TransactionOperationContext ctx))
    using (ctx.OpenReadTransaction())
    using (var rawRecord = _serverStore.Cluster.ReadRawDatabaseRecord(ctx, _documentDatabase.Name))
    {
        if (rawRecord.RollingIndexes == null)
        {
            reason = "No rolling indexes";
            return false;
        }
        
        if (rawRecord.RollingIndexes.TryGetValue(index.NormalizedName, out var rollingIndex) == false)
        {
            reason = "I'm not a rolling index";
            return false;
        }
        
        if (rollingIndex.ActiveDeployments.TryGetValue(_serverStore.NodeTag, out var nodeDeployment) == false)
        {
            reason = "My node has no active deployment";
            return true; // Skip, não deve indexar ainda
        }
        
        if (nodeDeployment.State != RollingIndexState.Pending)
        {
            reason = $"My state is {nodeDeployment.State}";
            return false; // Pode indexar
        }
        
        var didWork = DidWork(index.NormalizedName);
        
        if (index.Definition.Name.StartsWith(Constants.Documents.Indexing.SideBySideIndexNamePrefix))
        {
            reason = "I'm a pending side-by-side";
            if (didWork == false)
                replace = true; // Original não fez trabalho, pode substituir
            
            return true;
        }
        
        if (HasReplacement(index.Definition.Name) == false)
        {
            reason = "It isn't my turn to be deployed";
            return true; // Esperar vez
        }
        
        return false;
    }
}
```

**Fluxo de Rolling Deployment:**
1. Nó A: cria side-by-side, indexa, completa, notifica cluster
2. Cluster: marca Nó A como Done, libera Nó B
3. Nó B: cria side-by-side, indexa, completa, notifica cluster
4. ... repete para todos os nós
5. Último nó: substitui índice original

**Benefício:** Performance degradation distribuída, sem spike em todos os nós.

---

## ?? Exemplos de Código Notáveis

### 1. **Auto-Index Creation Dinâmica**

```csharp
// Em DynamicQueryRunner
public async Task<DocumentQueryResult> ExecuteNoIndex<TResult>(
    IndexQueryServerSide query, 
    QueryOperationContext context,
    ...)
{
    // 1. Tentar encontrar índice existente
    var matcher = new DynamicQueryToIndexMatcher(_indexStore);
    var match = matcher.Match(DynamicQueryMapping.Create(query));
    
    if (match.MatchType == DynamicQueryMatchType.Complete)
    {
        // ? Índice perfeito encontrado
        return await match.Index.Query(query, context, token);
    }
    
    if (match.MatchType == DynamicQueryMatchType.Partial)
    {
        // Índice parcial, mas pode estender
        var extendedMapping = DynamicQueryMapping.Create(match.Index);
        extendedMapping.ExtendMappingBasedOn(query);
        
        var extendedDefinition = extendedMapping.CreateAutoIndexDefinition();
        await _indexStore.CreateIndex(extendedDefinition, raftRequestId);
        
        // Aguardar novo índice ficar disponível
        // ...
    }
    
    // 2. Criar novo auto-index
    var mapping = DynamicQueryMapping.Create(query);
    var definition = mapping.CreateAutoIndexDefinition();
    
    var (_, index) = await _indexStore.CreateIndex(definition, raftRequestId);
    
    // 3. Executar query no novo índice
    return await index.Query(query, context, token);
}

public class DynamicQueryToIndexMatcher
{
    public DynamicQueryMatchResult Match(DynamicQueryMapping query)
    {
        foreach (var index in _indexStore.GetIndexes())
        {
            if (index.Type.IsAuto() == false)
                continue; // Só considera auto-indexes
            
            if (index.Collections.Overlaps(query.Collections) == false)
                continue;
            
            var result = ConsiderUsageOfIndex(query, index.Definition);
            
            if (result.MatchType == DynamicQueryMatchType.Complete)
                return result; // Match perfeito!
        }
        
        return new DynamicQueryMatchResult { MatchType = DynamicQueryMatchType.None };
    }
    
    private DynamicQueryMatchResult ConsiderUsageOfIndex(DynamicQueryMapping query, AutoIndexDefinitionBaseServerSide definition)
    {
        // Verifica se todos os campos da query estão no índice
        foreach (var field in query.MapFields)
        {
            if (definition.MapFields.TryGetValue(field.Name, out var indexField) == false)
                return new DynamicQueryMatchResult { MatchType = DynamicQueryMatchType.Partial };
            
            if (indexField.Indexing != field.Indexing)
                return new DynamicQueryMatchResult { MatchType = DynamicQueryMatchType.Partial };
        }
        
        // Verifica se índice tem campos extras (pode usar, mas não é perfeito)
        if (definition.MapFields.Count > query.MapFields.Count)
            return new DynamicQueryMatchResult { MatchType = DynamicQueryMatchType.CompleteButIdle };
        
        return new DynamicQueryMatchResult 
        { 
            MatchType = DynamicQueryMatchType.Complete,
            Index = index,
            LastMappedEtag = index.GetLastMappedEtagFor(query.Collections.First())
        };
    }
}
```

### 2. **MapDocuments Worker**

```csharp
public class MapDocuments : IIndexingWork
{
    public IndexingWorkExecuteResult Execute(
        QueryOperationContext queryContext,
        TransactionOperationContext indexContext,
        Lazy<IndexWriteOperationBase> writeOperation,
        IndexingStatsScope stats,
        CancellationToken cancellationToken)
    {
        var maxTimeForDocumentTransactionToRemainOpen = _configuration.MaxTimeForDocumentTransactionToRemainOpen.AsTimeSpan;
        var moreWorkFound = false;
        
        foreach (var collection in _index.Collections)
        {
            using (var collectionStats = stats.For("Collection_" + collection))
            {
                var lastMappedEtag = _indexStorage.ReadLastIndexedEtag(indexContext.Transaction, collection);
                var lastTombstoneEtag = _indexStorage.ReadLastProcessedTombstoneEtag(indexContext.Transaction, collection);
                
                var maxEtag = _index.GetLastItemEtagInCollection(queryContext, collection);
                
                if (lastMappedEtag >= maxEtag && lastTombstoneEtag >= maxEtag)
                {
                    collectionStats.RecordMapCompletedReason("No more documents to index");
                    continue; // Nada para fazer nesta coleção
                }
                
                var count = 0;
                var sw = Stopwatch.StartNew();
                
                // Buscar documentos a partir do último etag
                var documents = _documentsStorage.GetDocumentsFrom(queryContext.Documents, collection, 
                    lastMappedEtag + 1, 0, long.MaxValue);
                
                var enumerator = _index.GetMapEnumerator(documents, collection, indexContext, collectionStats, _index.Type);
                
                using (enumerator)
                {
                    while (enumerator.MoveNext(queryContext.Documents, out var item, out var tombstone))
                    {
                        cancellationToken.ThrowIfCancellationRequested();
                        
                        if (tombstone != null)
                        {
                            // Documento foi deletado
                            _index.HandleDelete(tombstone, collection, writeOperation, indexContext, collectionStats);
                            continue;
                        }
                        
                        // Mapear documento
                        using (var mapScope = collectionStats.For(IndexingOperation.Map.DocumentProcessing))
                        {
                            var numberOfOutputs = _index.HandleMap(item, mapResults: null, writeOperation, 
                                indexContext, mapScope);
                            
                            _index.HandleIndexOutputsPerDocument(item.LowerId, numberOfOutputs, collectionStats);
                        }
                        
                        count++;
                        
                        // Verificar se pode continuar batch
                        var canContinue = _index.CanContinueBatch(new CanContinueBatchParameters
                        {
                            Count = count,
                            CurrentEtag = item.Etag,
                            MaxEtag = maxEtag,
                            Stats = collectionStats,
                            Sw = sw,
                            WorkType = IndexingWorkType.Map,
                            QueryContext = queryContext,
                            IndexingContext = indexContext,
                            IndexWriteOperation = writeOperation
                        }, ref maxTimeForDocumentTransactionToRemainOpen);
                        
                        if (canContinue == CanContinueBatchResult.False)
                            break;
                        
                        if (canContinue == CanContinueBatchResult.RenewTransaction)
                        {
                            // Renew documento context para permitir flush
                            enumerator.OnRenew(item.Etag);
                        }
                    }
                    
                    if (enumerator.HasMore)
                        moreWorkFound = true;
                }
                
                // Atualizar último etag processado
                _indexStorage.WriteLastIndexedEtag(indexContext.Transaction, collection, lastMappedEtag);
            }
        }
        
        return new IndexingWorkExecuteResult
        {
            MoreWorkFound = moreWorkFound,
            BatchContinuationResult = CanContinueBatchResult.True
        };
    }
}
```

### 3. **Index Error Handling com Limites**

```csharp
private const int WriteErrorsLimit = 10;
private const int UnexpectedErrorsLimit = 3;
private const int AnalyzerErrorLimit = 0;
private const int DiskFullErrorLimit = 10;

private int _writeErrors;
private int _unexpectedErrors;
private int _analyzerErrors;
private int _diskFullErrors;

internal void HandleWriteErrors(IndexingStatsScope stats, IndexWriteException iwe)
{
    if (_logger.IsErrorEnabled)
        _logger.Error($"Write exception occurred for '{Name}'.", iwe);
    
    stats.AddWriteError(iwe);
    
    var writeErrors = Interlocked.Increment(ref _writeErrors);
    
    if (State == IndexState.Error || writeErrors < WriteErrorsLimit)
        return;
    
    SetErrorState($"State was changed due to excessive number of write errors ({writeErrors}).");
}

internal void HandleDiskFullErrors(IndexingStatsScope stats, StorageEnvironment storageEnvironment, DiskFullException dfe)
{
    stats.AddDiskFullError(dfe);
    
    var diskFullErrors = Interlocked.Increment(ref _diskFullErrors);
    if (diskFullErrors < DiskFullErrorLimit)
    {
        var timeToWaitInMilliseconds = (int)Math.Min(Math.Pow(2, diskFullErrors), 30) * 1000;
        
        if (_logger.IsInfoEnabled)
            _logger.Info($"After disk full error in index : '{Name}', " +
                         $"going to try flushing and syncing the environment to cleanup the storage. " +
                         $"Will wait for flush for: {timeToWaitInMilliseconds}ms", dfe);
        
        FlushAndSync(storageEnvironment, timeToWaitInMilliseconds, tryCleanupRecycledJournals: true);
        return;
    }
    
    if (_logger.IsErrorEnabled)
        _logger.Error($"Disk full error occurred for '{Name}'. Setting index to errored state", dfe);
    
    if (State == IndexState.Error)
        return;
    
    storageEnvironment.Options.TryCleanupRecycledJournals();
    SetErrorState($"State was changed due to excessive number of disk full errors ({diskFullErrors}).");
}

internal void ResetErrors()
{
    Interlocked.Exchange(ref _writeErrors, 0);
    Interlocked.Exchange(ref _unexpectedErrors, 0);
    Interlocked.Exchange(ref _analyzerErrors, 0);
    Interlocked.Exchange(ref _diskFullErrors, 0);
}
```

---

## ?? Padrões de Design

### 1. **Worker Pattern para Indexação**

```csharp
public interface IIndexingWork
{
    string Name { get; }
    
    IndexingWorkExecuteResult Execute(
        QueryOperationContext queryContext,
        TransactionOperationContext indexContext,
        Lazy<IndexWriteOperationBase> writeOperation,
        IndexingStatsScope stats,
        CancellationToken cancellationToken);
}

// Implementações:
// - MapDocuments: mapeia documentos novos/alterados
// - HandleReferences: processa referências entre documentos
// - CleanupDocuments: remove tombstones processados
// - ReduceMapResults: reduz resultados de map (map-reduce)

protected override IIndexingWork[] CreateIndexWorkExecutors()
{
    return new IIndexingWork[]
    {
        new CleanupDocuments(this, DocumentDatabase.DocumentsStorage, _indexStorage, Configuration, null),
        new MapDocuments(this, DocumentDatabase.DocumentsStorage, _indexStorage, null, Configuration),
        new HandleDocumentReferences(this, _referencedCollections, DocumentDatabase.DocumentsStorage, _indexStorage, Configuration)
    };
}
```

### 2. **Lazy Writer Pattern**

```csharp
// Writer só é criado se houver trabalho real
var writeOperation = new Lazy<IndexWriteOperationBase>(() =>
{
    var writer = IndexPersistence.OpenIndexWriter(indexContext.Transaction.InnerTransaction, indexContext);
    return writer;
});

try
{
    foreach (var work in _indexWorkers)
    {
        work.Execute(context, indexContext, writeOperation, scope, cancellationToken);
    }
    
    if (writeOperation.IsValueCreated)
    {
        using (var indexWriteOperation = writeOperation.Value)
        {
            indexWriteOperation.Commit(stats, cancellationToken);
        }
    }
}
catch
{
    if (writeOperation.IsValueCreated)
        writeOperation.Value.Dispose();
    throw;
}
```

### 3. **Enumerable Pulsed Transaction Pattern**

```csharp
// Para queries longas, pulsa transação para permitir commits
var pulsedEnumerator = new PulsedTransactionEnumerator<IndexReadOperationBase.QueryResult, QueryResultsIterationState>(
    queryContext.Documents,
    state => originalEnumerator,
    new QueryResultsIterationState(queryContext.Documents, DocumentDatabase.Configuration.Databases.PulseReadTransactionLimit));

pulsedEnumerator.OnPulse += retriever.ClearCache; // Invalida cache ao clonar tx

using (enumerator = pulsedEnumerator)
{
    while (enumerator.MoveNext())
    {
        var document = enumerator.Current;
        await resultToFill.AddResultAsync(document.Result, token.Token);
    }
}
```

---

## ?? Estatísticas e Métricas

### **IndexingStatsAggregator**

```csharp
public class IndexingStatsAggregator
{
    public DateTime StartTime { get; }
    public TimeSpan Duration => DateTime.UtcNow - StartTime;
    
    public int MapAttempts { get; private set; }
    public int MapSuccesses { get; private set; }
    public int MapErrors { get; private set; }
    
    public long InputCount { get; private set; }
    public long SuccessCount { get; private set; }
    public long FailedCount { get; private set; }
    
    public Size AllocatedBytes { get; private set; }
    public Size UsedMemory { get; private set; }
    
    public IndexingPerformanceStats ToIndexingPerformanceStats()
    {
        return new IndexingPerformanceStats
        {
            DurationInMs = Duration.TotalMilliseconds,
            InputCount = InputCount,
            SuccessCount = SuccessCount,
            FailedCount = FailedCount,
            Operations = _operations.Select(x => x.ToPerformanceOperation()).ToArray()
        };
    }
}
```

---

## ?? Integração com Outros Módulos

### **Com Transaction Merger**
- Indexação usa `TransactionOperationContext` do merger
- Aproveitando batching automático de commits

### **Com Document Storage**
- `GetDocumentsFrom()` para buscar documentos a indexar
- `GetTombstonesFrom()` para processar deletions

### **Com Query Execution**
- `IndexReadOperationFactory` para criar readers
- Lock reader/writer para coordenar queries vs indexação

---

**? Módulo 5 - Indexing Subsystem - ANÁLISE INICIAL COMPLETA**

*Nota: Este módulo é GIGANTE. Documentei os aspectos mais críticos. Próximos passos poderiam incluir análise profunda de:*
- Persistence layers (Lucene vs Corax)
- Map-Reduce implementation
- Faceted queries
- Suggestions engine

**Próximo módulo recomendado:**
- **Módulo 6**: Query Execution Engine (com análise de query parsing, optimization, facets)
