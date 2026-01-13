# Raven.Server - Módulo 6: Query Execution Engine

## ?? Visão Geral

O **Query Execution Engine** é responsável por receber queries RQL (Raven Query Language), determinar qual índice usar, e executar a query de forma otimizada. É o componente que conecta as requisições de clientes aos índices criados pelo Indexing Subsystem.

### Responsabilidades Principais

- **Query Routing**: Determinar qual runner usar (Static, Dynamic, Collection)
- **Index Selection**: Escolher melhor índice (existente ou criar auto-index)
- **Query Optimization**: Aplicar otimizações antes da execução
- **Result Retrieval**: Buscar documentos e aplicar projeções
- **Special Queries**: Facets, Suggestions, MoreLikeThis, Intersect
- **Stream Queries**: Queries com streaming de resultados

### Componentes do Módulo

```
Documents/Queries/
??? QueryRunner.cs                       # Entry point (~300 LOC)
??? AbstractDatabaseQueryRunner.cs       # Base abstrata
??? StaticIndexQueryRunner.cs            # Queries em índices estáticos
??? Dynamic/
?   ??? DynamicQueryRunner.cs            # Auto-index creation (~250 LOC)
?   ??? DynamicQueryMapping.cs           # Mapeamento de query ? index
?   ??? DynamicQueryToIndexMatcher.cs    # Algoritmo de match (~400 LOC)
?   ??? CollectionQueryRunner.cs         # Scan de coleção
??? Facets/
?   ??? FacetedQueryParser.cs            # Parse de facets
?   ??? FacetQuery.cs                    # Execução de facets
??? Suggestions/
?   ??? SuggestionQueryRunner.cs         # Suggestions
??? MoreLikeThis/
?   ??? MoreLikeThisQueryRunner.cs       # MoreLikeThis
??? FieldsToFetch.cs                     # Projeções e campos
??? QueryMetadata.cs                     # Metadata da query
??? IndexQueryServerSide.cs              # Query interna (vs client-side)
```

---

## ??? Arquitetura Interna

### 1. **QueryRunner - O Orquestrador**

```csharp
public sealed class QueryRunner : AbstractDatabaseQueryRunner
{
    private const int NumberOfRetries = 3;

    private readonly StaticIndexQueryRunner _static;
    private readonly AbstractDatabaseQueryRunner _dynamic;
    private readonly CollectionQueryRunner _collection;

    public QueryRunner(DocumentDatabase database) : base(database)
    {
        _static = new StaticIndexQueryRunner(database);
        
        _dynamic = database.Configuration.Indexing.DisableQueryOptimizerGeneratedIndexes
            ? new InvalidQueryRunner(database)  // ? Queries dinâmicas desabilitadas
            : new DynamicQueryRunner(database); // ? Auto-index creation habilitado
        
        _collection = new CollectionQueryRunner(database);
    }

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public AbstractDatabaseQueryRunner GetRunner(IndexQueryServerSide query)
    {
        if (query.Metadata.IsDynamic)
        {
            if (query.Metadata.IsCollectionQuery == false)
                return _dynamic;  // Query genérica, criar auto-index
            
            return _collection;   // Query simples de coleção, scan direto
        }
        
        return _static;           // Query em índice estático
    }

    public override async Task<DocumentQueryResult> ExecuteQuery(
        IndexQueryServerSide query, 
        QueryOperationContext queryContext, 
        long? existingResultEtag, 
        OperationCancelToken token)
    {
        Exception lastException = null;
        
        // ?? Retry logic: índices podem ser substituídos durante query!
        for (var i = 0; i < NumberOfRetries; i++)
        {
            try
            {
                Stopwatch sw = null;
                QueryTimingsScope scope;
                DocumentQueryResult result;
                
                using (scope = query.Timings?.Start())
                {
                    if (scope == null)
                        sw = Stopwatch.StartNew();
                    
                    result = await GetRunner(query).ExecuteQuery(query, queryContext, existingResultEtag, token);
                }
                
                result.DurationInMs = sw != null 
                    ? (long)sw.Elapsed.TotalMilliseconds 
                    : (long)scope.Duration.TotalMilliseconds;
                
                return result;
            }
            catch (ObjectDisposedException e)
            {
                if (Database.DatabaseShutdown.IsCancellationRequested)
                    throw;
                
                lastException = e;
                await WaitForIndexBeingLikelyReplacedDuringQuery();
            }
            catch (OperationCanceledException e)
            {
                if (Database.DatabaseShutdown.IsCancellationRequested)
                    throw;
                
                if (token.Token.IsCancellationRequested)
                    throw;
                
                lastException = e;
                await WaitForIndexBeingLikelyReplacedDuringQuery();
            }
        }
        
        throw CreateRetriesFailedException(lastException);
    }
    
    private Task WaitForIndexBeingLikelyReplacedDuringQuery()
    {
        // Aguarda 500ms para índice terminar de ser substituído
        return Task.Delay(500);
    }
}
```

**?? Insight:** Sistema de retry é essencial porque side-by-side index replacement pode ocorrer durante queries!

### 2. **DynamicQueryRunner - Auto-Index Magic**

```csharp
public sealed class DynamicQueryRunner : AbstractDatabaseQueryRunner
{
    private readonly IndexStore _indexStore;

    public override async Task<DocumentQueryResult> ExecuteQuery(
        IndexQueryServerSide query, 
        QueryOperationContext queryContext, 
        long? existingResultEtag, 
        OperationCancelToken token)
    {
        (long? Index, Index Instance) result;
        
        // 1. MATCH: Tentar encontrar índice existente ou criar novo
        using (query.Timings?.For(nameof(QueryTimingsScope.Names.Optimizer)))
            result = await MatchIndex(query, createAutoIndexIfNoMatchIsFound: true, null, token.Token);
        
        var index = result.Instance;
        queryContext.WithIndex(index);
        
        // 2. ETAG CHECK: Evitar recomputação se resultado não mudou
        if (query.Metadata.HasOrderByRandom == false && existingResultEtag.HasValue)
        {
            var etag = index.GetIndexEtag(queryContext, query.Metadata);
            if (etag == existingResultEtag)
                return DocumentQueryResult.NotModifiedResult;
        }
        
        // 3. EXECUTE: Executar query no índice selecionado
        using (QueryRunner.MarkQueryAsRunning(index.Name, query, token))
        {
            var queryResult = await index.Query(query, queryContext, token);
            queryResult.AutoIndexCreationRaftIndex = result.Index; // ? Incluir Raft index de criação
            
            return queryResult;
        }
    }

    public async Task<(long? Index, Index Instance)> MatchIndex(
        IndexQueryServerSide query, 
        bool createAutoIndexIfNoMatchIsFound, 
        TimeSpan? customStalenessWaitTimeout, 
        CancellationToken token)
    {
        // Fast path: query especifica auto-index por nome
        if (query.Metadata.AutoIndexName != null)
        {
            var index = GetIndex(query.Metadata.AutoIndexName, throwIfNotExists: false);
            
            if (index != null)
                return (null, index);
        }
        
        // Slow path: procurar melhor índice ou criar novo
        return await CreateAutoIndexIfNeeded(query, createAutoIndexIfNoMatchIsFound, customStalenessWaitTimeout, token);
    }
}
```

---

## ?? Fluxo de Execução

### **Query Execution - Pipeline Completo**

```
???????????????????????????????????????????????????????????????
?                   Query Arrives (RQL)                       ?
???????????????????????????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  1. Parse & Validate                  ?
        ?     - QueryMetadata creation          ?
        ?     - IsDynamic determination         ?
        ?     - IsCollectionQuery check         ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  2. Select Runner                     ?
        ?     Static / Dynamic / Collection     ?
        ????????????????????????????????????????
                           ?
                           ?
??????????????????????????????????????????????????????????????
?  3. DYNAMIC PATH (Auto-Index)                              ?
??????????????????????????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  3a. Create DynamicQueryMapping       ?
        ?      - Extract fields from WHERE      ?
        ?      - Extract fields from ORDER BY   ?
        ?      - Extract fields from SELECT     ?
        ?      - Extract spatial/vector fields  ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  3b. DynamicQueryToIndexMatcher       ?
        ?      Match loop over all indexes      ?
        ????????????????????????????????????????
                           ?
        ???????????????????????????????????????
        ?                                     ?
        ?                                     ?
????????????????                    ????????????????
? COMPLETE     ?                    ? PARTIAL      ?
? Match found  ?                    ? Extend index ?
????????????????                    ????????????????
        ?                                     ?
        ?                                     ?
        ?              ????????????????????????????????????????
        ?              ?  Create Extended Auto-Index          ?
        ?              ?  - Clone existing index definition   ?
        ?              ?  - Add missing fields                ?
        ?              ?  - Submit to Raft                    ?
        ?              ?  - Mark old index as superseded      ?
        ?              ????????????????????????????????????????
        ?                                     ?
        ???????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  3c. Wait for Non-Stale Results?      ?
        ?      (if WaitForNonStaleResults=true) ?
        ????????????????????????????????????????
                           ?
                           ?
??????????????????????????????????????????????????????????????
?  4. EXECUTE QUERY ON INDEX                                 ?
??????????????????????????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  4a. Open Index Reader                ?
        ?      (Lucene or Corax)                ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  4b. Execute Query                    ?
        ?      - Parse RQL to index query       ?
        ?      - Apply filters                  ?
        ?      - Apply sorting                  ?
        ?      - Apply paging (skip/take)       ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  4c. Retrieve Results                 ?
        ?      - Fetch matching doc IDs         ?
        ?      - Load documents                 ?
        ?      - Apply projections              ?
        ?      - Handle includes                ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  5. Post-Processing                   ?
        ?     - Highlightings                   ?
        ?     - Explanations                    ?
        ?     - Timings                         ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  6. Return DocumentQueryResult        ?
        ?     - Results                         ?
        ?     - TotalResults                    ?
        ?     - Includes                        ?
        ?     - IsStale                         ?
        ?     - IndexTimestamp                  ?
        ?     - ResultEtag                      ?
        ????????????????????????????????????????
```

---

## ?? Técnicas de Performance

### 1. **DynamicQueryToIndexMatcher - Algoritmo de Match**

**Problema:** Como escolher o melhor índice existente para uma query?

**Solução:** Sistema de scoring com múltiplos critérios.

```csharp
public DynamicQueryMatchResult Match(DynamicQueryMapping query, List<Explanation> explanations = null)
{
    var bestComplete = DynamicQueryMatchResult.Failure;
    
    foreach (var index in _indexStore.GetIndexesForCollection(query.ForCollection))
    {
        // 1. Verificar compatibilidade básica
        if (query.SearchEngineType != index.SearchEngineType)
            continue; // Corax vs Lucene não compatíveis
        
        if (query.IsGroupBy)
        {
            if (index.Type != IndexType.AutoMapReduce)
                continue;
        }
        else if (index.Type != IndexType.AutoMap)
            continue;
        
        var auto = (AutoIndexDefinitionBaseServerSide)index.Definition;
        
        // 2. Considerar uso deste índice
        var result = ConsiderUsageOfIndex(query, auto, explanations);
        
        // 3. Scoring: Decidir se é melhor que o atual
        string reason = null;
        bool hasBetterMatch = false;
        
        if (BetterMatchAvailable(out var bothMatch))
        {
            hasBetterMatch = result.MatchType > bestComplete.MatchType;
            reason = "A better match was available";
        }
        else if (result.LastMappedEtag != bestComplete.LastMappedEtag)
        {
            // ? CRITÉRIO 1: Preferir índice mais atualizado
            hasBetterMatch = result.LastMappedEtag > bestComplete.LastMappedEtag;
            reason = "Wasn't the most up to date index matching this query";
        }
        else if (bothMatch && result.MatchType != bestComplete.MatchType)
        {
            // ? CRITÉRIO 2: Preferir índice ativo vs idle
            hasBetterMatch = result.MatchType > bestComplete.MatchType;
            reason = "The index is idle. The preference is for active indexes";
        }
        else if (result.NumberOfMappedFields != bestComplete.NumberOfMappedFields)
        {
            // ? CRITÉRIO 3: Preferir índice mais específico
            hasBetterMatch = result.NumberOfMappedFields > bestComplete.NumberOfMappedFields;
            reason = "Wasn't the widest index matching this query";
        }
        
        if (hasBetterMatch)
            bestComplete = result;
        
        bool BetterMatchAvailable(out bool bothComplete)
        {
            bothComplete = false;
            if (result.MatchType == bestComplete.MatchType)
                return false;
            
            if (result.IsComplete() && bestComplete.IsComplete())
            {
                // Ambos são match perfeito, escolher por outros critérios
                bothComplete = true;
                return false;
            }
            
            return true;
        }
    }
    
    return bestComplete;
}

internal DynamicQueryMatchResult ConsiderUsageOfIndex(
    DynamicQueryMapping query, 
    AutoIndexDefinitionBaseServerSide definition, 
    List<Explanation> explanations = null)
{
    var collection = query.ForCollection;
    var indexName = definition.Name;
    
    // Verificações de compatibilidade
    if (definition.Collections.Contains(collection) == false)
    {
        explanations?.Add(new Explanation(indexName, 
            "Index does not apply to collection"));
        return new DynamicQueryMatchResult(indexName, DynamicQueryMatchType.Failure);
    }
    
    if (definition.Collections.Count > 1)
    {
        explanations?.Add(new Explanation(indexName, 
            "Index contains more than a single entity name"));
        return new DynamicQueryMatchResult(indexName, DynamicQueryMatchType.Failure);
    }
    
    var index = _indexStore.GetIndex(definition.Name);
    if (index == null)
        return new DynamicQueryMatchResult(definition.Name, DynamicQueryMatchType.Failure);
    
    var state = index.State;
    if (state == IndexState.Error || state == IndexState.Disabled || index.IsInvalidIndex())
    {
        explanations?.Add(new Explanation(indexName, 
            "Cannot do dynamic queries on disabled index or index with errors"));
        return new DynamicQueryMatchResult(indexName, DynamicQueryMatchType.Failure);
    }
    
    var currentBestState = DynamicQueryMatchType.Complete;
    
    // Verificar cada campo da query
    foreach (var field in query.MapFields.Values)
    {
        if (definition.TryGetField(field.Name, out var indexField))
        {
            // ? Campo existe, verificar características
            
            if (field.Vector != null)
            {
                if (field.Vector.Equals(indexField.Vector) == false)
                {
                    explanations?.Add(new Explanation(indexName, 
                        $"Field {indexField.Name} is not vector searchable"));
                    return new DynamicQueryMatchResult(indexName, DynamicQueryMatchType.Partial);
                }
                continue;
            }
            
            if (field.IsFullTextSearch && indexField.Indexing.HasFlag(AutoFieldIndexing.Search) == false)
            {
                explanations?.Add(new Explanation(indexName, 
                    $"Field {indexField.Name} is not searchable, while query needs search()"));
                return new DynamicQueryMatchResult(indexName, DynamicQueryMatchType.Partial);
            }
            
            if (field.HasHighlighting && indexField.Indexing.HasFlag(AutoFieldIndexing.Highlighting) == false)
            {
                explanations?.Add(new Explanation(indexName, 
                    $"Field {indexField.Name} does not have highlighting"));
                return new DynamicQueryMatchResult(indexName, DynamicQueryMatchType.Partial);
            }
            
            if (field.IsExactSearch && indexField.Indexing.HasFlag(AutoFieldIndexing.Exact) == false)
            {
                explanations?.Add(new Explanation(indexName, 
                    $"Field {indexField.Name} is not exactable, while query needs exact()"));
                return new DynamicQueryMatchResult(indexName, DynamicQueryMatchType.Partial);
            }
            
            if (field.Spatial != null)
            {
                if (field.Spatial.Equals(indexField.Spatial) == false)
                {
                    explanations?.Add(new Explanation(indexName, 
                        $"Field {indexField.Name} is not spatial"));
                    return new DynamicQueryMatchResult(indexName, DynamicQueryMatchType.Failure);
                }
            }
            
            if (field.HasSuggestions && indexField.HasSuggestions == false)
            {
                explanations?.Add(new Explanation(indexName, 
                    $"Field {indexField.Name} does not have suggestions enabled"));
                return new DynamicQueryMatchResult(indexName, DynamicQueryMatchType.Partial);
            }
        }
        else
        {
            // ? Campo não existe no índice
            explanations?.Add(new Explanation(indexName, $"Missing field: {field.Name}"));
            currentBestState = DynamicQueryMatchType.Partial;
        }
    }
    
    if (currentBestState == DynamicQueryMatchType.Complete && state == IndexState.Idle)
    {
        currentBestState = DynamicQueryMatchType.CompleteButIdle;
    }
    
    long lastMappedEtagFor = index.GetLastMappedEtagFor(collection);
    
    return new DynamicQueryMatchResult(indexName, currentBestState)
    {
        LastMappedEtag = lastMappedEtagFor,
        NumberOfMappedFields = definition.MapFields.Count
    };
}
```

**Resultado:** Sistema sofisticado de matching com 4 níveis de qualidade!

- **Complete**: Índice perfeito
- **CompleteButIdle**: Perfeito mas idle (será acordado)
- **Partial**: Faltam campos (será estendido)
- **Failure**: Incompatível

### 2. **Auto-Index Extension via Superseding**

**Problema:** Query precisa de campos que índice existente não tem. Criar novo índice do zero é lento.

**Solução:** Clonar índice existente e adicionar campos faltantes.

```csharp
private async Task<(long? Index, Index Instance)> CreateAutoIndexIfNeeded(
    IndexQueryServerSide query, 
    bool createAutoIndexIfNoMatchIsFound, 
    TimeSpan? customStalenessWaitTimeout, 
    CancellationToken token)
{
    var map = DynamicQueryMapping.Create(query, defaultAutoIndexingEngineType);
    
    (long? Index, Index Instance) result = default;
    
    while (TryMatchExistingIndexToQuery(map, out result.Instance) == false)
    {
        if (createAutoIndexIfNoMatchIsFound == false)
            throw new IndexDoesNotExistException("No Auto Index matching query");
        
        if (query.DisableAutoIndexCreation)
            throw new InvalidOperationException("Auto Index creation disabled");
        
        // ? Criar novo índice (possivelmente estendido de existente)
        var definition = map.CreateAutoIndexDefinition();
        result = await _indexStore.CreateIndex(definition, RaftIdGenerator.NewId());
        
        var index = result.Instance;
        if (index == null)
        {
            // Índice foi deletado, tentar encontrar melhor match
            continue;
        }
        
        // Configurar timeout padrão de staleness
        if (query.WaitForNonStaleResultsTimeout.HasValue == false)
        {
            query.WaitForNonStaleResultsTimeout = customStalenessWaitTimeout ?? TimeSpan.FromSeconds(15);
        }
        
        // ?? Cleanup: Deletar índices superseded em background
        var t = CleanupSupersededAutoIndexes(index, map, RaftIdGenerator.NewId(), token)
            .ContinueWith(task =>
            {
                if (task.Exception != null && token.IsCancellationRequested == false)
                {
                    if (_indexStore.Logger.IsInfoEnabled)
                        _indexStore.Logger.Info($"Failed to delete superseded indexes for {index.Name}");
                }
            }, token);
        
        if (query.WaitForNonStaleResults && 
            Database.Configuration.Indexing.TimeToWaitBeforeDeletingAutoIndexMarkedAsIdle.AsTimeSpan == TimeSpan.Zero)
        {
            await t; // Usado em testes
        }
        
        break;
    }
    
    return result;
}

private bool TryMatchExistingIndexToQuery(DynamicQueryMapping map, out Index index)
{
    var dynamicQueryToIndex = new DynamicQueryToIndexMatcher(_indexStore);
    
    var matchResult = dynamicQueryToIndex.Match(map);
    
    switch (matchResult.MatchType)
    {
        case DynamicQueryMatchType.Complete:
        case DynamicQueryMatchType.CompleteButIdle:
            index = GetIndex(matchResult.IndexName, throwIfNotExists: false);
            if (index == null)
            {
                // Auto-index foi deletado, tentar novamente
                break;
            }
            
            return true;
        
        case DynamicQueryMatchType.Partial:
            // ? EXTEND: Clonar índice e adicionar campos
            var currentIndex = _indexStore.GetIndex(matchResult.IndexName);
            if (currentIndex != null)
            {
                if (map.SupersededIndexes == null)
                    map.SupersededIndexes = new List<Index>();
                
                map.SupersededIndexes.Add(currentIndex);
                
                // Clone definition e adiciona campos
                map.ExtendMappingBasedOn((AutoIndexDefinitionBaseServerSide)currentIndex.Definition);
            }
            
            break;
    }
    
    index = null;
    return false;
}
```

**Exemplo:**

```
Query inicial:
  WHERE Name = 'John'
  
? Cria Auto/Users/ByName
  - Name (Exact)

Query subsequente:
  WHERE Name = 'John' AND Age > 30
  
? Match parcial com Auto/Users/ByName (falta Age)
? Estende para Auto/Users/ByNameAndAge
  - Name (Exact)
  - Age (Range)
? Marca Auto/Users/ByName como superseded
? Deleta Auto/Users/ByName após indexação completa
```

**Benefício:** Reutiliza trabalho de indexação já feito!

### 3. **Superseded Index Cleanup**

```csharp
private async Task CleanupSupersededAutoIndexes(
    Index index, 
    DynamicQueryMapping map, 
    string raftRequestId, 
    CancellationToken token)
{
    if (map.SupersededIndexes == null || map.SupersededIndexes.Count == 0)
        return;
    
    while (token.IsCancellationRequested == false)
    {
        AsyncManualResetEvent.FrozenAwaiter indexingBatchCompleted;
        try
        {
            indexingBatchCompleted = index.GetIndexingBatchAwaiter();
        }
        catch (ObjectDisposedException)
        {
            break; // Novo índice foi deletado
        }
        
        // Verificar se novo índice alcançou etag dos antigos
        var maxSupersededEtag = 0L;
        foreach (var supersededIndex in map.SupersededIndexes)
        {
            try
            {
                var etag = supersededIndex.GetLastMappedEtagFor(map.ForCollection);
                maxSupersededEtag = Math.Max(etag, maxSupersededEtag);
            }
            catch (OperationCanceledException)
            {
                // Índice superseded já foi deletado
            }
        }
        
        long currentEtag;
        try
        {
            currentEtag = index.GetLastMappedEtagFor(map.ForCollection);
        }
        catch (OperationCanceledException)
        {
            break; // Novo índice foi deletado
        }
        
        if (currentEtag >= maxSupersededEtag)
        {
            // ? Novo índice alcançou etag dos antigos, pode deletar!
            
            // Aguardar timeout para drenar queries pendentes
            var timeout = Database.Configuration.Indexing.TimeBeforeDeletionOfSupersededAutoIndex.AsTimeSpan;
            if (timeout != TimeSpan.Zero)
            {
                await TimeoutManager.WaitFor(timeout).ConfigureAwait(false);
            }
            
            // Deletar índices superseded
            foreach (var supersededIndex in map.SupersededIndexes)
            {
                try
                {
                    await _indexStore.DeleteIndex(supersededIndex.Name, $"{raftRequestId}/{supersededIndex.Name}");
                }
                catch (IndexDoesNotExistException)
                {
                    // Já foi deletado
                }
            }
            
            break;
        }
        
        // Aguardar próximo batch de indexação
        if (await indexingBatchCompleted.WaitAsync() == false)
            break;
    }
}
```

### 4. **Query Retry Logic**

**Problema:** Side-by-side index replacement pode causar `ObjectDisposedException` durante query.

**Solução:** Retry automático com delay.

```csharp
public override async Task<DocumentQueryResult> ExecuteQuery(...)
{
    Exception lastException = null;
    
    for (var i = 0; i < NumberOfRetries; i++)
    {
        try
        {
            return await GetRunner(query).ExecuteQuery(query, queryContext, existingResultEtag, token);
        }
        catch (ObjectDisposedException e)
        {
            if (Database.DatabaseShutdown.IsCancellationRequested)
                throw;
            
            lastException = e;
            
            // ?? Aguardar 500ms para replacement terminar
            await WaitForIndexBeingLikelyReplacedDuringQuery();
        }
        catch (OperationCanceledException e)
        {
            if (Database.DatabaseShutdown.IsCancellationRequested)
                throw;
            
            if (token.Token.IsCancellationRequested)
                throw;
            
            lastException = e;
            await WaitForIndexBeingLikelyReplacedDuringQuery();
        }
    }
    
    throw CreateRetriesFailedException(lastException);
}

private Task WaitForIndexBeingLikelyReplacedDuringQuery()
{
    return Task.Delay(500);
}
```

**Benefício:** Transparência para cliente. Query sucede mesmo durante replacement!

### 5. **Etag-Based Result Caching**

**Problema:** Cliente pode ter resultado cached. Como evitar retransmitir?

**Solução:** Result etag baseado em index etag + query metadata.

```csharp
public override async Task<DocumentQueryResult> ExecuteQuery(...)
{
    var index = result.Instance;
    queryContext.WithIndex(index);
    
    // ? Check etag antes de executar
    if (query.Metadata.HasOrderByRandom == false && existingResultEtag.HasValue)
    {
        var etag = index.GetIndexEtag(queryContext, query.Metadata);
        if (etag == existingResultEtag)
            return DocumentQueryResult.NotModifiedResult; // 304 Not Modified
    }
    
    using (QueryRunner.MarkQueryAsRunning(index.Name, query, token))
    {
        var queryResult = await index.Query(query, queryContext, token);
        queryResult.AutoIndexCreationRaftIndex = result.Index;
        
        return queryResult;
    }
}

// Index.cs
protected virtual unsafe long CalculateIndexEtag(
    QueryOperationContext queryContext,
    TransactionOperationContext indexContext, 
    QueryMetadata q, 
    bool isStale)
{
    var length = MinimumSizeForCalculateIndexEtagLength(q);
    var indexEtagBytes = stackalloc byte[length];
    
    foreach (var collection in Collections)
    {
        var lastDocEtag = GetLastItemEtagInCollection(queryContext, collection);
        var lastTombstoneEtag = GetLastTombstoneEtagInCollection(queryContext, collection);
        var lastMappedEtag = _indexStorage.ReadLastIndexedEtag(indexContext.Transaction, collection);
        var lastProcessedTombstoneEtag = _indexStorage.ReadLastProcessedTombstoneEtag(indexContext.Transaction, collection);
        
        *(long*)indexEtagBytes = lastDocEtag;
        indexEtagBytes += sizeof(long);
        *(long*)indexEtagBytes = lastTombstoneEtag;
        indexEtagBytes += sizeof(long);
        *(long*)indexEtagBytes = lastMappedEtag;
        indexEtagBytes += sizeof(long);
        *(long*)indexEtagBytes = lastProcessedTombstoneEtag;
        indexEtagBytes += sizeof(long);
    }
    
    *(int*)indexEtagBytes = Definition.GetHashCode();
    indexEtagBytes += sizeof(int);
    *indexEtagBytes = isStale ? (byte)0 : (byte)1;
    indexEtagBytes += sizeof(byte);
    *indexEtagBytes = (byte)State;
    indexEtagBytes += sizeof(byte);
    *(long*)indexEtagBytes = _indexStorage.CreatedTimestampAsBinary;
    
    return (long)Hashing.XXHash64.Calculate(indexEtagBytes, (ulong)length);
}
```

**Ganho:** Economia massiva de CPU/network quando resultado não mudou!

---

## ?? Exemplos de Código Notáveis

### 1. **DynamicQueryMapping - Extração de Campos**

```csharp
public static DynamicQueryMapping Create(IndexQueryServerSide query, SearchEngineType searchEngineType)
{
    var result = new DynamicQueryMapping
    {
        ForCollection = query.Metadata.CollectionName,
        IsGroupBy = query.Metadata.IsGroupBy,
        SearchEngineType = searchEngineType
    };
    
    // Extrair campos do WHERE
    foreach (var field in query.Metadata.WhereFields.Values)
    {
        var fieldName = field.Name;
        
        var autoField = new AutoIndexField
        {
            Name = fieldName,
            Indexing = AutoFieldIndexing.Default
        };
        
        if (field.IsFullTextSearch)
            autoField.Indexing |= AutoFieldIndexing.Search;
        
        if (field.IsExactSearch)
            autoField.Indexing |= AutoFieldIndexing.Exact;
        
        if (field.HasHighlighting)
            autoField.Indexing |= AutoFieldIndexing.Highlighting;
        
        if (field.Spatial != null)
            autoField.Spatial = field.Spatial;
        
        if (field.Vector != null)
            autoField.Vector = field.Vector;
        
        if (field.HasSuggestions)
            autoField.HasSuggestions = true;
        
        result.MapFields[fieldName] = autoField;
    }
    
    // Extrair campos do ORDER BY
    if (query.Metadata.OrderBy != null)
    {
        foreach (var sortedField in query.Metadata.OrderBy)
        {
            if (sortedField.OrderingType == OrderByFieldType.Random)
                continue;
            
            if (sortedField.OrderingType == OrderByFieldType.Score)
                continue;
            
            var fieldName = sortedField.Name;
            
            if (result.MapFields.ContainsKey(fieldName) == false)
            {
                result.MapFields[fieldName] = new AutoIndexField
                {
                    Name = fieldName,
                    Indexing = AutoFieldIndexing.Default
                };
            }
        }
    }
    
    // Extrair campos do SELECT (projeções)
    if (query.Metadata.SelectFields != null)
    {
        foreach (var selectField in query.Metadata.SelectFields)
        {
            // Projeções não afetam índice (apenas resultados)
        }
    }
    
    // GROUP BY
    if (query.Metadata.GroupBy != null)
    {
        foreach (var groupByField in query.Metadata.GroupBy)
        {
            result.GroupByFields[groupByField.Name] = new AutoIndexField
            {
                Name = groupByField.Name,
                Aggregation = groupByField.AggregationOperation,
                GroupByArrayBehavior = groupByField.GroupByArrayBehavior
            };
        }
    }
    
    return result;
}
```

### 2. **FieldsToFetch - Projection Handling**

```csharp
public sealed class FieldsToFetch
{
    public string[] Fields { get; }
    public bool IsProjection { get; }
    public bool IsDistinct { get; }
    
    public FieldsToFetch(IndexQueryServerSide query, IndexDefinitionBaseServerSide indexDefinition, IndexType indexType)
    {
        if (query.Metadata.SelectFields == null || query.Metadata.SelectFields.Length == 0)
        {
            // Sem projeção, retornar documento completo
            IsProjection = false;
            Fields = null;
            return;
        }
        
        IsProjection = true;
        IsDistinct = query.Metadata.IsDistinct;
        
        var fields = new List<string>();
        
        foreach (var selectField in query.Metadata.SelectFields)
        {
            if (selectField.Function != null)
            {
                // Função de agregação (COUNT, SUM, etc)
                fields.Add(selectField.Alias ?? selectField.Function.Name);
            }
            else if (selectField.SourceAlias != null)
            {
                // Load/Include
                fields.Add(selectField.Alias ?? selectField.Name);
            }
            else if (selectField.IsGroupByKey)
            {
                // Campo de group by
                fields.Add(selectField.Name);
            }
            else
            {
                // Campo regular
                fields.Add(selectField.Name);
            }
        }
        
        Fields = fields.ToArray();
    }
    
    public Document ProjectFromDocument(Document document)
    {
        if (IsProjection == false)
            return document;
        
        var projected = new DynamicJsonValue();
        
        foreach (var field in Fields)
        {
            if (document.Data.TryGet(field, out object value))
            {
                projected[field] = value;
            }
        }
        
        return new Document
        {
            Id = document.Id,
            Data = document.Data.Context.ReadObject(projected, document.Id)
        };
    }
}
```

### 3. **CollectionQueryRunner - Scan sem Índice**

```csharp
public sealed class CollectionQueryRunner : AbstractDatabaseQueryRunner
{
    public override async Task<DocumentQueryResult> ExecuteQuery(...)
    {
        var collection = query.Metadata.CollectionName;
        var fieldsToFetch = new FieldsToFetch(query, null, IndexType.None);
        
        using (var queryScope = query.Timings?.For(nameof(QueryTimingsScope.Names.Query)))
        {
            var totalResults = new Reference<long>();
            var scannedResults = new Reference<int>();
            var skippedResults = new Reference<long>();
            
            // ? Scan direto da coleção (sem índice)
            var documents = new CollectionQueryEnumerable(
                Database,
                Database.DocumentsStorage,
                fieldsToFetch,
                collection,
                query,
                queryScope,
                queryContext,
                includeDocumentsCommand,
                includeRevisionsCommand,
                includeCompareExchangeValuesCommand,
                totalResults,
                scannedResults,
                skippedResults,
                token.Token);
            
            var result = new DocumentQueryResult();
            
            await foreach (var document in documents)
            {
                if (query.Limit.HasValue && result.Results.Count >= query.Limit.Value)
                    break;
                
                result.Results.Add(document);
            }
            
            result.TotalResults = totalResults.Value;
            result.ScannedResults = scannedResults.Value;
            result.SkippedResults = skippedResults.Value;
            
            return result;
        }
    }
}
```

---

## ?? Padrões de Design

### 1. **Strategy Pattern - Query Runners**

```csharp
// Base abstrata
public abstract class AbstractDatabaseQueryRunner
{
    protected readonly DocumentDatabase Database;
    
    public abstract Task<DocumentQueryResult> ExecuteQuery(...);
    public abstract Task ExecuteStreamQuery(...);
    public abstract Task<SuggestionQueryResult> ExecuteSuggestionQuery(...);
    // ... outros métodos
}

// Implementações concretas
public class StaticIndexQueryRunner : AbstractDatabaseQueryRunner { ... }
public class DynamicQueryRunner : AbstractDatabaseQueryRunner { ... }
public class CollectionQueryRunner : AbstractDatabaseQueryRunner { ... }
public class InvalidQueryRunner : AbstractDatabaseQueryRunner { ... } // Para quando auto-index está desabilitado

// Seleção em runtime
public AbstractDatabaseQueryRunner GetRunner(IndexQueryServerSide query)
{
    if (query.Metadata.IsDynamic)
    {
        if (query.Metadata.IsCollectionQuery == false)
            return _dynamic;
        
        return _collection;
    }
    
    return _static;
}
```

### 2. **Template Method Pattern - Index Matching**

```csharp
public async Task<(long? Index, Index Instance)> MatchIndex(...)
{
    // Template method define fluxo
    
    // 1. Fast path: nome específico
    if (query.Metadata.AutoIndexName != null)
    {
        var index = GetIndex(query.Metadata.AutoIndexName, throwIfNotExists: false);
        if (index != null)
            return (null, index);
    }
    
    // 2. Slow path: buscar/criar
    return await CreateAutoIndexIfNeeded(...);
}

private async Task<(long? Index, Index Instance)> CreateAutoIndexIfNeeded(...)
{
    var map = DynamicQueryMapping.Create(query, searchEngineType);
    
    // Loop até encontrar match
    while (TryMatchExistingIndexToQuery(map, out result.Instance) == false)
    {
        // Criar novo índice
        var definition = map.CreateAutoIndexDefinition();
        result = await _indexStore.CreateIndex(definition, raftRequestId);
        
        // Cleanup superseded
        await CleanupSupersededAutoIndexes(...);
        
        break;
    }
    
    return result;
}
```

### 3. **Chain of Responsibility - Field Extraction**

```csharp
public static DynamicQueryMapping Create(IndexQueryServerSide query, SearchEngineType searchEngineType)
{
    var result = new DynamicQueryMapping();
    
    // Chain de extratores
    ExtractWhereFields(query, result);
    ExtractOrderByFields(query, result);
    ExtractGroupByFields(query, result);
    ExtractSelectFields(query, result);
    
    return result;
}

private static void ExtractWhereFields(IndexQueryServerSide query, DynamicQueryMapping result)
{
    foreach (var field in query.Metadata.WhereFields.Values)
    {
        // Processar campo WHERE
    }
}

private static void ExtractOrderByFields(IndexQueryServerSide query, DynamicQueryMapping result)
{
    if (query.Metadata.OrderBy != null)
    {
        foreach (var sortedField in query.Metadata.OrderBy)
        {
            // Processar campo ORDER BY
        }
    }
}
```

---

## ?? Estatísticas e Métricas

### **QueryTimingsScope**

```csharp
public class QueryTimingsScope : IDisposable
{
    public DateTime Start { get; }
    public TimeSpan Duration => DateTime.UtcNow - Start;
    
    public Dictionary<string, QueryTimingsScope> Scopes { get; }
    
    public QueryTimingsScope For(string name, bool start = true)
    {
        if (Scopes.TryGetValue(name, out var scope) == false)
        {
            scope = new QueryTimingsScope { Name = name };
            Scopes[name] = scope;
        }
        
        if (start)
            scope.Start();
        
        return scope;
    }
    
    public void Dispose()
    {
        // Record duration
    }
}

// Uso:
using (var queryScope = query.Timings?.For(nameof(QueryTimingsScope.Names.Query)))
{
    using (queryScope?.For(nameof(QueryTimingsScope.Names.Optimizer)))
    {
        // Index selection logic
    }
    
    using (queryScope?.For(nameof(QueryTimingsScope.Names.Lucene)))
    {
        // Lucene query execution
    }
}
```

---

## ?? Integração com Outros Módulos

### **Com Indexing Subsystem**
- `Index.Query()` para executar queries
- `Index.GetIndexEtag()` para caching
- `IndexStore.CreateIndex()` para auto-indexes

### **Com Document Storage**
- `DocumentsStorage.GetDocumentsFrom()` para collection scans
- `DocumentsStorage.GetDocument()` para load de documentos

### **Com Transaction Merger**
- Usa `QueryOperationContext` que internamente pode usar merger
- Queries são read-only, não passam pelo merger

---

## ?? Lições Aprendidas

### 1. **Auto-Index Extension é Mais Eficiente que Criação**
- Clonar índice existente e adicionar campos reutiliza trabalho
- Indexação incremental a partir do etag do índice antigo
- Cleanup de índices superseded é assíncrono e não bloqueia

### 2. **Retry Logic é Essencial em Ambiente com Replacement**
- Side-by-side pode causar `ObjectDisposedException`
- Retry com delay permite replacement terminar
- Transparente para cliente

### 3. **Index Selection é Multi-Critério**
- Etag (atualidade)
- Estado (active vs idle)
- Número de campos (especificidade)
- Ordem importa!

### 4. **Etag-Based Caching Economiza Recursos**
- Index etag combina múltiplos fatores
- Cliente pode cachear resultado
- 304 Not Modified evita retransmissão

---

**? Módulo 6 - Query Execution Engine - COMPLETO**

*Próximo módulo recomendado:*
- **Módulo 7**: Replication Engine (como queries distribuídas funcionam em cluster)
