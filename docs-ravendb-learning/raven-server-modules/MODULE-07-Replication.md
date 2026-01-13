# Raven.Server - Módulo 7: Replication Engine

## ?? Visão Geral

O **Replication Engine** é responsável por replicar dados entre nós do cluster RavenDB, garantindo alta disponibilidade e consistência eventual. É um dos módulos mais complexos devido à necessidade de coordenar múltiplas conexões, resolver conflitos e manter change vectors sincronizados.

### Responsabilidades Principais

- **Outgoing Replication**: Enviar dados para nós de destino
- **Incoming Replication**: Receber dados de nós de origem
- **Conflict Resolution**: Resolver conflitos de atualização
- **Change Vector Management**: Rastrear versões em ambiente distribuído
- **Pull/Push Replication**: Suporte a replicação bidirecional
- **Hub Replication**: Replicação filtrada com controle de acesso

### Componentes do Módulo

```
Documents/Replication/
??? ReplicationLoader.cs                     # Orquestrador principal (~1.900 LOC!)
??? Outgoing/
?   ??? DatabaseOutgoingReplicationHandler.cs # Envio de dados
?   ??? OutgoingExternalReplicationHandler.cs
?   ??? OutgoingInternalReplicationHandler.cs
?   ??? OutgoingPullReplicationHandlerAsHub.cs
??? Incoming/
?   ??? IncomingReplicationHandler.cs         # Recebimento de dados
?   ??? IncomingPullReplicationHandler.cs
?   ??? ReplicationInitialHandshake.cs
??? ConflictManager.cs                        # Resolução de conflitos
??? ConflictResolver.cs                       # Estratégias de resolução
??? ChangeVectorUtils.cs                      # Manipulação de change vectors
??? Stats/                                    # Métricas de replicação
    ??? OutgoingReplicationStatsAggregator.cs
    ??? IncomingReplicationStatsAggregator.cs
```

---

## ??? Arquitetura Interna

### 1. **ReplicationLoader - O Orquestrador**

```csharp
public class ReplicationLoader : AbstractReplicationLoader<DocumentsContextPool, DocumentsOperationContext>
{
    // Gerenciamento de conexões outgoing
    private readonly ConcurrentSet<DatabaseOutgoingReplicationHandler> _outgoing = new();
    private readonly ConcurrentDictionary<ReplicationNode, ConnectionShutdownInfo> _outgoingFailureInfo = new();
    private readonly ConcurrentSet<ConnectionShutdownInfo> _reconnectQueue = new();
    
    // Gerenciamento de conexões incoming
    private readonly ConcurrentDictionary<IncomingConnectionInfo, DateTime> _incomingLastActivityTime = new();
    private readonly ConcurrentDictionary<IncomingConnectionInfo, ConcurrentQueue<IncomingConnectionRejectionInfo>> _incomingRejectionStats = new();
    
    // Configuração de destinos
    private readonly ConcurrentBag<ReplicationNode> _internalDestinations = new(); // Nós do cluster
    private readonly HashSet<ExternalReplicationBase> _externalDestinations = new(); // Replicação externa
    private List<ReplicationNode> _destinations = new();
    
    // Conflict resolution
    public ResolveConflictOnReplicationConfigurationChange ConflictResolver;
    public ConflictSolver ConflictSolverConfig;
    
    // Tracking de último etag enviado
    private readonly ConcurrentDictionary<ReplicationNode, LastEtagPerDestination> _lastSendEtagPerDestination = new();
    
    // Controle de lifecycle
    private readonly CancellationToken _shutdownToken;
    private readonly Timer _reconnectAttemptTimer;
    
    // Events
    public event Action<IncomingReplicationHandler> IncomingReplicationAdded;
    public event Action<IncomingReplicationHandler> IncomingReplicationRemoved;
    public event Action<DatabaseOutgoingReplicationHandler> OutgoingReplicationAdded;
    public event Action<DatabaseOutgoingReplicationHandler> OutgoingReplicationRemoved;
}
```

### 2. **Change Vector - Versionamento Distribuído**

```csharp
// Change Vector Format: "A:8-databaseId1, A:9-databaseId2"
// Cada entrada é: "Tag:Etag-DatabaseId"

public static class ChangeVectorUtils
{
    public static long GetEtagById(string changeVector, string dbId)
    {
        if (string.IsNullOrEmpty(changeVector))
            return 0;
        
        var entries = changeVector.Split(',');
        foreach (var entry in entries)
        {
            var parts = entry.Trim().Split(':');
            if (parts.Length != 2)
                continue;
            
            var etagAndDbId = parts[1].Split('-');
            if (etagAndDbId.Length != 2)
                continue;
            
            if (etagAndDbId[1] == dbId)
                return long.Parse(etagAndDbId[0]);
        }
        
        return 0;
    }
    
    public static string MergeVectors(string first, string second)
    {
        // Merge dois change vectors mantendo o maior etag de cada database
        var result = new Dictionary<string, (string Tag, long Etag)>();
        
        ProcessVector(first);
        ProcessVector(second);
        
        var sb = new StringBuilder();
        var isFirst = true;
        
        foreach (var (dbId, (tag, etag)) in result.OrderBy(x => x.Key))
        {
            if (!isFirst)
                sb.Append(", ");
            
            sb.Append(tag);
            sb.Append(':');
            sb.Append(etag);
            sb.Append('-');
            sb.Append(dbId);
            
            isFirst = false;
        }
        
        return sb.ToString();
        
        void ProcessVector(string cv)
        {
            if (string.IsNullOrEmpty(cv))
                return;
            
            var entries = cv.Split(',');
            foreach (var entry in entries)
            {
                var parts = entry.Trim().Split(':');
                var tag = parts[0];
                var etagAndDbId = parts[1].Split('-');
                var etag = long.Parse(etagAndDbId[0]);
                var dbId = etagAndDbId[1];
                
                if (!result.TryGetValue(dbId, out var existing) || existing.Etag < etag)
                {
                    result[dbId] = (tag, etag);
                }
            }
        }
    }
}
```

### 3. **Conflict Detection & Resolution**

```csharp
public class ConflictManager
{
    public enum ConflictStatus
    {
        Update,         // Pode atualizar normalmente
        Conflict,       // Conflito detectado
        AlreadyMerged   // Já foi mesclado anteriormente
    }
    
    public ConflictStatus GetConflictStatus(
        string incomingChangeVector,
        string existingChangeVector)
    {
        if (string.IsNullOrEmpty(existingChangeVector))
            return ConflictStatus.Update; // Documento novo
        
        if (string.IsNullOrEmpty(incomingChangeVector))
            return ConflictStatus.Conflict; // Sem change vector
        
        // Comparar vectors
        var comparison = ChangeVector.CompareVectors(incomingChangeVector, existingChangeVector);
        
        return comparison switch
        {
            ConflictStatus.Update => ConflictStatus.Update,
            ConflictStatus.Conflict => ConflictStatus.Conflict,
            ConflictStatus.AlreadyMerged => ConflictStatus.AlreadyMerged,
            _ => throw new ArgumentOutOfRangeException()
        };
    }
    
    public void ResolveConflict(
        DocumentsOperationContext context,
        string docId,
        Document incoming,
        Document existing,
        ConflictSolverConfig config)
    {
        ScriptResolver scriptResolver = null;
        
        if (config?.ResolveByCollection != null &&
            config.ResolveByCollection.TryGetValue(incoming.Collection, out var script))
        {
            scriptResolver = new ScriptResolver(script);
        }
        
        Document resolved;
        
        if (scriptResolver != null)
        {
            // Usar script de resolução customizado
            resolved = scriptResolver.Resolve(docId, incoming, existing, context);
        }
        else
        {
            // Estratégia padrão: usar documento mais recente
            var incomingEtag = ChangeVectorUtils.GetEtagById(incoming.ChangeVector, incoming.DbId);
            var existingEtag = ChangeVectorUtils.GetEtagById(existing.ChangeVector, existing.DbId);
            
            resolved = incomingEtag > existingEtag ? incoming : existing;
        }
        
        // Merge change vectors
        resolved.ChangeVector = ChangeVectorUtils.MergeVectors(
            incoming.ChangeVector,
            existing.ChangeVector);
        
        // Salvar documento resolvido
        context.DocumentDatabase.DocumentsStorage.Put(context, docId, null, resolved.Data, resolved.ChangeVector);
    }
}
```

---

## ?? Fluxo de Replicação

### **Outgoing Replication - Pipeline Completo**

```
???????????????????????????????????????????????????????????????
?              Outgoing Replication Handler                   ?
???????????????????????????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  1. Connect to Destination            ?
        ?     - Fetch TCP connection info       ?
        ?     - Establish TCP connection        ?
        ?     - Send initial handshake          ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  2. Initial Handshake                 ?
        ?     - Send database ID                ?
        ?     - Send current change vector      ?
        ?     - Receive last accepted etag      ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  3. Collect Changes (Loop)            ?
        ?     - Documents (lastEtag + 1...)     ?
        ?     - Tombstones                      ?
        ?     - Attachments                     ?
        ?     - Counters                        ?
        ?     - Time Series                     ?
        ?     - Revisions                       ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  4. Build Replication Batch           ?
        ?     - Serialize items                 ?
        ?     - Apply filtering (if any)        ?
        ?     - Compress batch                  ?
        ?     - Calculate batch hash            ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  5. Send Batch                        ?
        ?     - Write batch header              ?
        ?     - Write batch items               ?
        ?     - Flush network stream            ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  6. Receive Acknowledgement           ?
        ?     - Read ACK from destination       ?
        ?     - Update last accepted etag       ?
        ?     - Update last sent change vector  ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  7. Update Metrics                    ?
        ?     - Documents per second            ?
        ?     - Bytes per second                ?
        ?     - Last heartbeat                  ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  8. Delay (if configured)             ?
        ?     - Wait DelayReplicationFor        ?
        ?     - Prevents instant replication    ?
        ????????????????????????????????????????
                           ?
                           ?
                           ??????????? Loop
                                     ?
                                     ?
```

### **Incoming Replication - Pipeline Completo**

```
???????????????????????????????????????????????????????????????
?              Incoming Replication Handler                   ?
???????????????????????????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  1. Accept Connection                 ?
        ?     - Validate certificate            ?
        ?     - Read initial handshake          ?
        ?     - Send last accepted etag         ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  2. Receive Batch (Loop)              ?
        ?     - Read batch header               ?
        ?     - Read batch items                ?
        ?     - Decompress if needed            ?
        ?     - Verify batch hash               ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  3. Apply Changes to Storage          ?
        ?     For each item in batch:           ?
        ?     ?? Document ? Put or Delete       ?
        ?     ?? Tombstone ? Record deletion    ?
        ?     ?? Attachment ? Store blob        ?
        ?     ?? Counter ? Merge values         ?
        ?     ?? Time Series ? Add segments     ?
        ?     ?? Revision ? Store history       ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  4. Conflict Detection                ?
        ?     For each change:                  ?
        ?     ?? Get existing document          ?
        ?     ?? Compare change vectors         ?
        ?     ?? Detect conflict                ?
        ?     ?? Resolve if needed              ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  5. Send Acknowledgement              ?
        ?     - Calculate new change vector     ?
        ?     - Send ACK to source              ?
        ?     - Update last processed etag      ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  6. Update Metrics                    ?
        ?     - Documents received              ?
        ?     - Bytes received                  ?
        ?     - Last activity time              ?
        ????????????????????????????????????????
                           ?
                           ?
                           ??????????? Loop
                                     ?
                                     ?
```

---

## ?? Técnicas de Performance

### 1. **Connection Management com Retry Exponencial**

**Problema:** Conexões de replicação podem falhar. Como reconectar sem sobrecarregar?

**Solução:** Retry exponencial com backoff e limite máximo.

```csharp
public sealed class ConnectionShutdownInfo
{
    public ReplicationNode Node { get; set; }
    public int RetriesCount { get; private set; }
    public DateTime RetryOn { get; private set; }
    public string DestinationDbId { get; set; }
    public long LastHeartbeatTicks { get; set; }
    public ConcurrentQueue<Exception> Errors { get; } = new();
    
    private readonly double _maxConnectionTimeout; // Ex: 60 segundos
    
    public void OnError(Exception e)
    {
        Errors.Enqueue(e);
        
        // Manter apenas últimos 10 erros
        while (Errors.Count > 10)
            Errors.TryDequeue(out _);
        
        RetriesCount++;
        
        // Exponencial backoff: 2^n segundos
        var timeout = Math.Min(
            Math.Pow(2, RetriesCount), 
            _maxConnectionTimeout / 1000.0);
        
        RetryOn = DateTime.UtcNow.AddSeconds(timeout);
    }
    
    public void Reset()
    {
        RetriesCount = 0;
        RetryOn = DateTime.MinValue;
        
        while (Errors.TryDequeue(out _))
        {
            // Clear queue
        }
    }
}

// Reconnect timer (a cada 15 segundos por padrão)
private readonly Timer _reconnectAttemptTimer;

private void ForceTryReconnectAll()
{
    if (_reconnectQueue.Count == 0)
        return;
    
    if (Interlocked.CompareExchange(ref _reconnectInProgress, 1, 0) == 1)
        return; // Já está reconectando
    
    try
    {
        foreach (var failure in _reconnectQueue)
        {
            if (failure.RetryOn > DateTime.UtcNow)
            {
                _reconnectQueue.Add(failure); // Re-enqueue para próxima tentativa
                continue;
            }
            
            if (_reconnectQueue.TryRemove(failure) == false)
                continue;
            
            if (_outgoingFailureInfo.Values.Contains(failure) == false)
                continue; // Conexão não existe mais
            
            try
            {
                AddAndStartOutgoingReplication(failure.Node);
            }
            catch (Exception e)
            {
                if (_logger.IsErrorEnabled)
                    _logger.Error($"Failed to start outgoing replication to {failure.Node}", e);
            }
        }
    }
    finally
    {
        Interlocked.Exchange(ref _reconnectInProgress, 0);
    }
}
```

**Resultado:** Reconexão automática com pressão gradual, evitando storm de conexões!

### 2. **Batching de Mudanças**

**Problema:** Enviar cada mudança individualmente é ineficiente.

**Solução:** Coletar mudanças em lotes e enviar em batch.

```csharp
public class ReplicationBatch
{
    public List<ReplicationBatchItem> Items { get; } = new();
    public int MaxSize { get; set; } = 16 * 1024; // 16 KB padrão
    
    public bool TryAdd(ReplicationBatchItem item)
    {
        if (Items.Count >= MaxSize)
            return false;
        
        Items.Add(item);
        return true;
    }
    
    public void Clear()
    {
        Items.Clear();
    }
}

public async Task SendBatch(ReplicationBatch batch)
{
    using (var stream = new MemoryStream())
    {
        // 1. Serializar batch
        using (var writer = new BinaryWriter(stream, Encoding.UTF8, leaveOpen: true))
        {
            writer.Write(batch.Items.Count);
            
            foreach (var item in batch.Items)
            {
                writer.Write((byte)item.Type);
                
                switch (item.Type)
                {
                    case ReplicationBatchItem.ReplicationItemType.Document:
                        WriteDocument(writer, (DocumentReplicationItem)item);
                        break;
                    
                    case ReplicationBatchItem.ReplicationItemType.Attachment:
                        WriteAttachment(writer, (AttachmentReplicationItem)item);
                        break;
                    
                    // ... outros tipos
                }
            }
        }
        
        // 2. Comprimir batch (opcional)
        if (stream.Length > 1024) // Comprimir se > 1KB
        {
            using (var compressed = new MemoryStream())
            {
                using (var gzip = new GZipStream(compressed, CompressionMode.Compress, leaveOpen: true))
                {
                    stream.Position = 0;
                    await stream.CopyToAsync(gzip);
                }
                
                // Usar compressed se for menor
                if (compressed.Length < stream.Length * 0.9)
                {
                    stream.SetLength(0);
                    compressed.Position = 0;
                    await compressed.CopyToAsync(stream);
                }
            }
        }
        
        // 3. Enviar batch
        stream.Position = 0;
        await _networkStream.WriteAsync(stream.ToArray());
        await _networkStream.FlushAsync();
    }
}
```

### 3. **Pull Replication com Filtered Paths**

**Problema:** Como replicar apenas subconjunto de documentos para hub externo?

**Solução:** Filtro baseado em prefixo de IDs.

```csharp
public class PullReplicationParams
{
    public string Name { get; set; }
    public string[] AllowedPaths { get; set; } // Ex: ["users/", "orders/"]
    public PullReplicationMode Mode { get; set; }
    public PreventDeletionsMode? PreventDeletionsMode { get; set; }
}

public bool ShouldReplicateItem(ReplicationBatchItem item, string[] allowedPaths)
{
    if (allowedPaths == null || allowedPaths.Length == 0)
        return true; // Sem filtro
    
    var docId = item.Id.ToString().ToLowerInvariant();
    
    foreach (var path in allowedPaths)
    {
        if (docId.StartsWith(path.ToLowerInvariant()))
            return true; // Match!
    }
    
    return false; // Não passa no filtro
}

// Aplicar filtro ao coletar mudanças
foreach (var item in collectedChanges)
{
    if (ShouldReplicateItem(item, _pullReplicationParams.AllowedPaths))
    {
        batch.TryAdd(item);
    }
}
```

### 4. **Write Assurance (Confirmação de Escrita)**

**Problema:** Como garantir que mudança foi replicada para maioria dos nós?

**Solução:** Aguardar ACKs de N nós antes de confirmar para cliente.

```csharp
public async Task<int> WaitForReplicationAsync(
    DocumentsOperationContext context, 
    int numberOfReplicasToWaitFor, 
    TimeSpan waitForReplicasTimeout, 
    ChangeVector lastChangeVector)
{
    lastChangeVector = lastChangeVector.StripTrxnTags(context);
    
    var sp = Stopwatch.StartNew();
    
    while (true)
    {
        var internalDestinations = _internalDestinations.Select(x => x.Url).ToHashSet();
        var waitForNextReplicationAsync = WaitForNextReplicationAsync();
        
        // Contar quantos nós já replicaram além deste change vector
        var past = ReplicatedPastInternalDestinations(context, internalDestinations, lastChangeVector);
        
        if (past >= numberOfReplicasToWaitFor)
            return past; // ? Sucesso!
        
        var remaining = waitForReplicasTimeout - sp.Elapsed;
        if (remaining < TimeSpan.Zero)
            return past; // ?? Timeout
        
        var timeout = TimeoutManager.WaitFor(remaining);
        
        try
        {
            if (await Task.WhenAny(waitForNextReplicationAsync, timeout) == timeout)
                return past; // Timeout
        }
        catch (OperationCanceledException)
        {
            return past; // Cancelado
        }
    }
}

private int ReplicatedPastInternalDestinations(
    DocumentsOperationContext context, 
    HashSet<string> internalUrls, 
    ChangeVector changeVector)
{
    var count = 0;
    
    foreach (var destination in _outgoing)
    {
        if (internalUrls.Contains(destination.Destination.Url) == false)
            continue;
        
        var conflictStatus = Database.DocumentsStorage.GetConflictStatusForOrder(
            context, 
            changeVector, 
            destination.LastAcceptedChangeVector);
        
        if (conflictStatus == ConflictStatus.AlreadyMerged)
            count++; // Este nó já tem esta mudança ou mais recente
    }
    
    return count;
}

// Calcular número mínimo de réplicas (maioria)
public int GetMinNumberOfReplicas()
{
    return (_numberOfSiblings + 1) / 2; // Quórum
}
```

**Exemplo:**
```
Cluster com 5 nós: mínimo = (5 + 1) / 2 = 3 réplicas
Cluster com 3 nós: mínimo = (3 + 1) / 2 = 2 réplicas
```

### 5. **Tombstone Cleanup com Replicação**

**Problema:** Quando deletar tombstones sem perder informação de replicação?

**Solução:** Só deletar tombstones após todos os destinos processarem.

```csharp
public long GetMinimalEtagForReplication(
    Dictionary<string, LastTombstoneInfo> lastProcessedTombstonesInfo = null, 
    string collection = null)
{
    long minEtag = long.MaxValue;
    
    // 1. Verificar external replications
    using (_server.ContextPool.AllocateOperationContext(out TransactionOperationContext ctx))
    using (ctx.OpenReadTransaction())
    {
        var dbRecord = _server.Cluster.ReadRawDatabaseRecord(ctx, Database.Name);
        var externals = dbRecord.ExternalReplications;
        
        if (externals != null)
        {
            foreach (var external in externals)
            {
                var state = GetExternalReplicationState(_server, Database.Name, external.TaskId, ctx);
                var myEtag = ChangeVectorUtils.GetEtagById(state.SourceChangeVector, Database.DbBase64Id);
                
                minEtag = Math.Min(myEtag, minEtag);
                
                AddOrUpdateLastEtag(lastProcessedTombstonesInfo, collection, external.Name, myEtag, 
                    ITombstoneAware.TombstoneDeletionBlockerType.ExternalReplication);
            }
        }
    }
    
    // 2. Verificar internal replications
    var replicationNodes = new List<ReplicationNode>();
    
    foreach (var lastEtagPerDestination in _lastSendEtagPerDestination)
    {
        minEtag = Math.Min(lastEtagPerDestination.Value.LastEtag, minEtag);
        
        if (lastProcessedTombstonesInfo != null)
        {
            switch (lastEtagPerDestination.Key)
            {
                case PullReplicationAsSink pullReplicationAsSink:
                    AddOrUpdateLastEtag(lastProcessedTombstonesInfo, collection, pullReplicationAsSink.Name, 
                        lastEtagPerDestination.Value.LastEtag, 
                        ITombstoneAware.TombstoneDeletionBlockerType.PullReplicationAsSink);
                    break;
                
                case InternalReplication internalReplication:
                    AddOrUpdateLastEtag(lastProcessedTombstonesInfo, collection, internalReplication.NodeTag, 
                        lastEtagPerDestination.Value.LastEtag, 
                        ITombstoneAware.TombstoneDeletionBlockerType.InternalReplication);
                    break;
            }
        }
    }
    
    // 3. Se faltam destinos, não pode deletar nada
    if (replicationNodes.Count > 0)
    {
        if (lastProcessedTombstonesInfo == null)
            return 0; // Não sabe estado de todos, não pode deletar
        
        foreach (var node in replicationNodes)
        {
            AddOrUpdateLastEtag(lastProcessedTombstonesInfo, collection, ((InternalReplication)node).NodeTag, 0, 
                ITombstoneAware.TombstoneDeletionBlockerType.InternalReplication);
        }
        
        return 0;
    }
    
    return minEtag; // ? Pode deletar tombstones até este etag
}
```

---

## ?? Exemplos de Código Notáveis

### 1. **Dynamic Topology Changes**

```csharp
public void HandleDatabaseRecordChange(DatabaseRecord newRecord, long index)
{
    HandleConflictResolverChange(newRecord, index);
    HandleTopologyChange(newRecord);
    UpdateConnectionStrings(newRecord);
}

private void HandleTopologyChange(DatabaseRecord newRecord)
{
    var instancesToDispose = new List<IDisposable>();
    
    if (newRecord == null || _server.IsPassive() || _replicationDisabledByMarker)
    {
        // Cluster sendo desligado, dropar todas as conexões
        DropOutgoingConnections(Destinations, instancesToDispose);
        DropIncomingConnections(Destinations, instancesToDispose);
        
        _internalDestinations.Clear();
        _externalDestinations.Clear();
        _destinations.Clear();
        
        DisposeConnections(instancesToDispose);
        return;
    }
    
    _clusterTopology = GetClusterTopology();
    
    HandleReplicationChanges(newRecord, instancesToDispose);
    
    // Rebuild lista de destinos
    var destinations = new List<ReplicationNode>();
    destinations.AddRange(_internalDestinations);
    destinations.AddRange(_externalDestinations);
    _destinations = destinations;
    
    _numberOfSiblings = _destinations.Select(x => x.Url)
        .Intersect(_clusterTopology.AllNodes.Select(x => x.Value))
        .Count();
    
    DisposeConnections(instancesToDispose);
}

private void HandleInternalReplication(DatabaseRecord newRecord, List<IDisposable> instancesToDispose)
{
    var newInternalDestinations = newRecord.Topology?.GetDestinations(
        _server.NodeTag, 
        Database.Name, 
        newRecord.DeletionInProgress, 
        _clusterTopology, 
        _server.Engine.CurrentState);
    
    var changes = DatabaseTopology.FindChanges(_internalDestinations, newInternalDestinations);
    
    if (changes.RemovedDestiantions.Count > 0)
    {
        var removed = changes.RemovedDestiantions.Select(r => new InternalReplication
        {
            NodeTag = _clusterTopology.TryGetNodeTagByUrl(r).NodeTag,
            Url = r,
            Database = Database.Name
        }).ToList();
        
        DropOutgoingConnections(removed, instancesToDispose);
        DropIncomingConnections(removed, instancesToDispose);
    }
    
    if (changes.AddedDestinations.Count > 0)
    {
        var added = changes.AddedDestinations.Select(r => new InternalReplication
        {
            NodeTag = _clusterTopology.TryGetNodeTagByUrl(r).NodeTag,
            Url = r,
            Database = Database.Name
        }).ToList();
        
        _ = Task.Run(() =>
        {
            try
            {
                StartOutgoingConnections(added);
            }
            catch (Exception e)
            {
                if (_logger.IsErrorEnabled)
                    _logger.Error($"Failed to start outgoing connections to {added.Count} new destinations", e);
            }
        });
    }
    
    _internalDestinations.Clear();
    
    if (newInternalDestinations != null)
    {
        foreach (var item in newInternalDestinations)
            _internalDestinations.Add(item);
    }
}
```

---

## ?? Padrões de Design

### 1. **Event-Driven Architecture**

```csharp
// Eventos para monitoramento e integração
public event Action<IncomingReplicationHandler> IncomingReplicationAdded;
public event Action<IncomingReplicationHandler> IncomingReplicationRemoved;
public event Action<DatabaseOutgoingReplicationHandler> OutgoingReplicationAdded;
public event Action<DatabaseOutgoingReplicationHandler> OutgoingReplicationRemoved;
public event Action<LiveReplicationPulsesCollector.ReplicationPulse> OutgoingReplicationConnectionFailed;

// Uso:
outgoingReplication.Failed += OnOutgoingSendingFailed;
outgoingReplication.SuccessfulTwoWaysCommunication += OnOutgoingSendingSucceeded;
outgoingReplication.SuccessfulReplication += ResetReplicationFailuresInfo;
```

### 2. **Strategy Pattern - Conflict Resolution**

```csharp
public interface IConflictResolver
{
    Document Resolve(string docId, Document incoming, Document existing, DocumentsOperationContext context);
}

public class ScriptResolver : IConflictResolver
{
    private readonly string _script;
    
    public Document Resolve(string docId, Document incoming, Document existing, DocumentsOperationContext context)
    {
        // Executar script JavaScript customizado
        using (var scriptRunner = new ScriptRunner(context.Database, context.Database.Configuration, true))
        {
            scriptRunner.AddScript(_script);
            
            var result = scriptRunner.Run(context, new
            {
                id = docId,
                incoming = incoming.Data,
                existing = existing.Data
            });
            
            return new Document
            {
                Id = docId,
                Data = result,
                ChangeVector = ChangeVectorUtils.MergeVectors(incoming.ChangeVector, existing.ChangeVector)
            };
        }
    }
}

public class LatestWinsResolver : IConflictResolver
{
    public Document Resolve(string docId, Document incoming, Document existing, DocumentsOperationContext context)
    {
        var incomingEtag = ChangeVectorUtils.GetEtagById(incoming.ChangeVector, incoming.DbId);
        var existingEtag = ChangeVectorUtils.GetEtagById(existing.ChangeVector, existing.DbId);
        
        return incomingEtag > existingEtag ? incoming : existing;
    }
}
```

---

## ?? Estatísticas e Métricas

```csharp
public class OutgoingReplicationStatsAggregator
{
    public long NumberOfDocumentsSent { get; private set; }
    public long NumberOfAttachmentsSent { get; private set; }
    public long NumberOfCountersSent { get; private set; }
    public long NumberOfTimeSeriesSent { get; private set; }
    public Size TotalBytesSent { get; private set; }
    
    public double? GetProcessedPerSecondRate()
    {
        if (Duration.TotalSeconds == 0)
            return null;
        
        return NumberOfDocumentsSent / Duration.TotalSeconds;
    }
}

public class IncomingReplicationStatsAggregator
{
    public long NumberOfDocumentsReceived { get; private set; }
    public long NumberOfAttachmentsReceived { get; private set; }
    public long NumberOfConflictsDetected { get; private set; }
    public long NumberOfConflictsResolved { get; private set; }
}
```

---

## ?? Integração com Outros Módulos

### **Com Transaction Merger**
- Replicação usa `TransactionOperationContext` do merger
- Incoming changes passam pelo merger para batching

### **Com Document Storage**
- `DocumentsStorage.Put()` para aplicar mudanças
- `GetConflictStatusForOrder()` para detectar conflitos

### **Com Clustering (Rachis)**
- Topologia vem do cluster state
- Mudanças de configuração via Raft

---

## ?? Lições Aprendidas

### 1. **Change Vectors são Essenciais**
- Versionamento distribuído sem servidor central
- Detecção automática de conflitos
- Merge de históricos de mudanças

### 2. **Retry Exponencial Previne Storms**
- Backoff exponencial com limite
- Fila de reconexão assíncrona
- Metrics de falhas para observabilidade

### 3. **Batching Melhora Throughput**
- Reduz overhead de rede
- Compressão de batches grandes
- Balanceamento entre latência e throughput

### 4. **Tombstone Cleanup é Crítico**
- Só deletar após todos os destinos processarem
- Previne re-criação de documentos deletados
- Coordenação com disabled destinations

---

**? Módulo 7 - Replication Engine - COMPLETO**

*Próximo módulo recomendado:*
- **Módulo 8**: Clustering & Raft (como decisões são tomadas em cluster)
