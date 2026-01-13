# Raven.Server - Módulo 9: Subscriptions System

## ?? Visão Geral

O **Subscriptions System** é responsável por entregar mudanças de documentos em tempo real para clientes. É um sistema de **push-based data delivery** que garante **at-least-once delivery** e permite **processamento confiável** de mudanças.

### Responsabilidades Principais

- **Real-time Push**: Enviar mudanças para clientes conectados
- **At-Least-Once Delivery**: Garantir que mudanças sejam entregues
- **Batch Processing**: Agrupar documentos em lotes eficientes
- **Failover Handling**: Reconexão automática em falhas
- **Progress Tracking**: Rastreamento de progresso com change vectors

### Componentes do Módulo

```
Documents/Subscriptions/
??? SubscriptionStorage.cs                  # Armazenamento e gerenciamento
??? TcpHandlers/
?   ??? SubscriptionConnection.cs           # Conexão TCP com cliente
??? Processor/
?   ??? DocumentsDatabaseSubscriptionProcessor.cs
?   ??? RevisionsDatabaseSubscriptionProcessor.cs
??? Stats/
?   ??? SubscriptionBatchStatsAggregator.cs # Métricas
??? SubscriptionConnectionsState.cs         # Estado de conexões
```

---

## ??? Arquitetura Interna

### 1. **SubscriptionStorage - Gerenciamento Central**

```csharp
public class SubscriptionStorage : AbstractSubscriptionStorage<SubscriptionConnectionsState>
{
    internal readonly DocumentDatabase _db;
    
    // Events para notificações
    public event Action<string> OnAddTask;
    public event Action<string> OnRemoveTask;
    public event Action<SubscriptionConnection> OnEndConnection;
    public event Action<string, SubscriptionBatchStatsAggregator> OnEndBatch;
    
    // Armazenamento de estados de conexões
    private ConcurrentDictionary<long, SubscriptionConnectionsState> _subscriptions;
    
    public async Task<(long Index, long SubscriptionId)> PutSubscription(
        SubscriptionCreationOptions options, 
        string raftRequestId, 
        long? subscriptionId = null, 
        bool? disabled = false, 
        string mentor = null)
    {
        // Criar comando Raft
        var command = new PutSubscriptionCommand(_databaseName, options.Query, mentor, raftRequestId)
        {
            InitialChangeVector = options.ChangeVector,
            SubscriptionName = options.Name,
            SubscriptionId = subscriptionId,
            PinToMentorNode = options.PinToMentorNode,
            Disabled = disabled ?? false,
            ArchivedDataProcessingBehavior = options.ArchivedDataProcessingBehavior
        };
        
        // Enviar para líder Raft
        var (etag, _) = await _serverStore.SendToLeaderAsync(command);
        
        // Aguardar confirmação
        await _db.RachisLogIndexNotifications.WaitForIndexNotification(etag, _serverStore.Engine.OperationTimeout);
        
        if (subscriptionId != null)
            return (etag, subscriptionId.Value); // Update
        
        RaiseNotificationForTaskAdded(options.Name);
        
        return (etag, etag); // Create
    }
}
```

### 2. **SubscriptionConnection - Conexão com Cliente**

```csharp
public class SubscriptionConnection : SubscriptionConnectionBase<DatabaseIncludesCommandImpl>
{
    private readonly DocumentDatabase _database;
    public long CurrentBatchId;
    protected SubscriptionConnectionsState State;
    
    // Lifecycle da conexão
    public async Task RunAsync()
    {
        State = GetSubscriptionConnectionState();
        
        // 1. Setup do processor
        Processor = CreateProcessor(this);
        AfterProcessorCreation();
        
        // 2. Loop principal
        while (!CancellationToken.IsCancellationRequested)
        {
            try
            {
                // 3. Processar batch
                var result = await Processor.GetBatchAsync();
                
                // 4. Enviar para cliente
                await SendBatchAsync(result);
                
                // 5. Aguardar ACK
                await WaitForClientAckAsync();
                
                // 6. Atualizar progresso
                await OnClientAckAsync(clientReplyChangeVector);
            }
            catch (Exception ex)
            {
                HandleError(ex);
            }
        }
    }
}
```

### 3. **Subscription Query Parsing**

```csharp
public struct ParsedSubscription
{
    public string Collection;      // Coleção a monitorar
    public string Script;           // Script de filtro/projeção
    public string[] Functions;      // Funções auxiliares
    public bool Revisions;          // Incluir revisões?
    public string[] Includes;       // Includes de documentos
    public string[] CounterIncludes; // Includes de counters
    internal TimeSeriesIncludesField TimeSeriesIncludes; // Includes de time series
}

public static ParsedSubscription ParseSubscriptionQuery(string query)
{
    var queryParser = new QueryParser();
    queryParser.Init(query);
    var q = queryParser.Parse();
    
    // Validações
    if (q.IsDistinct)
        throw new NotSupportedException("Subscription does not support distinct queries");
    if (q.From.Index)
        throw new NotSupportedException("Subscription must specify a collection to use");
    if (q.GroupBy != null)
        throw new NotSupportedException("Subscription cannot specify a group by clause");
    if (q.OrderBy != null)
        throw new NotSupportedException("Subscription cannot specify an order by clause");
    
    // Extrair coleção
    var collectionName = q.From.From.FieldValue;
    
    // Parse de includes
    if (q.Include != null)
    {
        foreach (var include in q.Include)
        {
            // Processar includes de documentos, counters, time series
        }
    }
    
    // Gerar script JavaScript se houver WHERE ou SELECT
    if (q.Where == null && q.Select == null && q.SelectFunctionBody.FunctionText == null)
    {
        return new ParsedSubscription
        {
            Collection = collectionName,
            Revisions = revisions,
            Includes = includes?.ToArray(),
            CounterIncludes = counterIncludes?.ToArray(),
            TimeSeriesIncludes = timeSeriesIncludes
        };
    }
    
    // Gerar script JavaScript
    var writer = new StringWriter();
    
    if (q.From.Alias != null)
    {
        writer.Write("var ");
        writer.Write(q.From.Alias);
        writer.WriteLine(" = this;");
    }
    
    if (q.Where != null)
    {
        writer.Write("if (");
        new JavascriptCodeQueryVisitor(writer.GetStringBuilder(), q).VisitExpression(q.Where);
        writer.WriteLine(" )");
        writer.WriteLine("{");
    }
    
    if (q.Select != null)
    {
        writer.WriteLine(" return ");
        new JavascriptCodeQueryVisitor(writer.GetStringBuilder(), q).VisitExpression(q.Select[0].Expression);
        writer.WriteLine(";");
    }
    else
    {
        writer.WriteLine(" return true;");
    }
    
    if (q.Where != null)
        writer.WriteLine("}");
    
    var script = writer.GetStringBuilder().ToString();
    
    // Verificar sintaxe JavaScript
    new Parser(DefaultParserOptions).ParseScript(script);
    
    return new ParsedSubscription
    {
        Collection = collectionName,
        Script = script,
        Functions = q.DeclaredFunctions?.Values?.Select(x => x.FunctionText).ToArray() ?? Array.Empty<string>(),
        Includes = includes?.ToArray(),
        CounterIncludes = counterIncludes?.ToArray()
    };
}
```

---

## ?? Fluxo de Subscription

### **End-to-End Flow**

```
???????????????????????????????????????????????????????????????
?                Cliente Cria Subscription                    ?
?   client.Subscriptions.CreateAsync(new SubscriptionCreationOptions())
???????????????????????????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  1. PutSubscription no Server         ?
        ?     - Parse query                     ?
        ?     - Criar comando Raft              ?
        ?     - Replicar para cluster           ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  2. Subscription Armazenada           ?
        ?     - State persisted no Raft         ?
        ?     - SubscriptionId gerado           ?
        ?     - Nó responsável determinado      ?
        ????????????????????????????????????????
                           ?
                           ?
???????????????????????????????????????????????????????????????
?              Cliente Abre Worker Connection                 ?
?   var worker = store.Subscriptions.GetSubscriptionWorker<Order>("Orders")
???????????????????????????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  3. Estabelecer Conexão TCP           ?
        ?     - Cliente ? Server (TCP)          ?
        ?     - Handshake                       ?
        ?     - SubscriptionConnectionOptions   ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  4. Server Aceita Conexão             ?
        ?     - Validar permissões              ?
        ?     - Criar SubscriptionConnection    ?
        ?     - Iniciar SubscriptionProcessor   ?
        ????????????????????????????????????????
                           ?
                           ?
??????????????????????????????????????????????????????????????
?                 LOOP DE PROCESSAMENTO                      ?
??????????????????????????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  5. Processor.GetBatchAsync()         ?
        ?     - Ler documentos desde lastCV     ?
        ?     - Aplicar filtro/projeção         ?
        ?     - Incluir relacionados            ?
        ?     - Limitar batch size (4096 docs)  ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  6. SendBatchAsync()                  ?
        ?     - Serializar batch                ?
        ?     - Enviar via TCP                  ?
        ?     - Marcar como "pending ACK"       ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  7. Cliente Processa Batch            ?
        ?     await worker.Run(batch => {       ?
        ?         foreach (var item in batch.Items)
        ?             ProcessItem(item);        ?
        ?     })                                ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  8. Cliente Envia ACK                 ?
        ?     - ChangeVector do último item     ?
        ?     - Enviado via TCP                 ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  9. Server Processa ACK               ?
        ?     - AcknowledgeBatchAsync()         ?
        ?     - Atualizar lastCV no Raft        ?
        ?     - Liberar batch                   ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  10. Server Envia Confirm             ?
        ?     - Confirma processamento ACK      ?
        ?     - Cliente pode processar próximo  ?
        ????????????????????????????????????????
                           ?
                           ?
                           ??????????? Loop continua
                                     ?
                                     ?
```

---

## ?? Técnicas de Performance

### 1. **Batching Inteligente**

**Problema:** Enviar cada documento individualmente é ineficiente.

**Solução:** Agrupar documentos em batches.

```csharp
public class SubscriptionBatcher
{
    private const int DefaultBatchSize = 4096;
    private const int MaxBatchSizeInBytes = 32 * 1024 * 1024; // 32 MB
    
    public async Task<SubscriptionBatchResult> GetBatchAsync(
        DocumentsOperationContext context,
        string collection,
        string lastChangeVector,
        CancellationToken token)
    {
        var batch = new List<Document>();
        var totalSize = 0;
        var lastCV = lastChangeVector;
        
        // Iterar documentos desde lastChangeVector
        var iterator = _db.DocumentsStorage.GetDocumentsFrom(
            context, 
            collection, 
            lastChangeVector, 
            start: 0, 
            pageSize: DefaultBatchSize);
        
        foreach (var doc in iterator)
        {
            token.ThrowIfCancellationRequested();
            
            // Aplicar filtro se houver
            if (_patch != null && !ApplyFilter(doc))
                continue;
            
            batch.Add(doc);
            totalSize += doc.Data.Size;
            lastCV = doc.ChangeVector;
            
            // Verificar limites
            if (batch.Count >= DefaultBatchSize || 
                totalSize >= MaxBatchSizeInBytes)
            {
                break;
            }
        }
        
        return new SubscriptionBatchResult
        {
            CurrentBatch = batch,
            LastChangeVectorSentInThisBatch = lastCV
        };
    }
}
```

**Ganho:** Reduz overhead de rede e RTT. Throughput aumenta 10-100x.

### 2. **At-Least-Once Delivery Guarantee**

**Problema:** Como garantir que mudanças sejam entregues mesmo com falhas?

**Solução:** Rastreamento persistente de progresso via change vectors.

```csharp
public async Task AcknowledgeBatchAsync(long batchId, string clientReplyChangeVector)
{
    // 1. Criar comando Raft para atualizar progresso
    var command = new AcknowledgeSubscriptionBatchCommand(
        _databaseName,
        _subscriptionId,
        _subscriptionName,
        clientReplyChangeVector,
        batchId,
        Guid.NewGuid().ToString());
    
    // 2. Replicar para cluster (persistência durável)
    var (index, _) = await _serverStore.SendToLeaderAsync(command);
    
    // 3. Aguardar commit
    await _db.RachisLogIndexNotifications.WaitForIndexNotification(
        index, 
        _serverStore.Engine.OperationTimeout);
    
    // 4. Atualizar estado local
    _state.LastChangeVectorSent = ChangeVectorUtils.MergeVectors(
        _state.LastChangeVectorSent,
        clientReplyChangeVector);
}
```

**Resultado:**
- ? Se cliente crashar antes de ACK ? Re-enviar batch na reconexão
- ? Se servidor crashar após ACK ? Progresso persistido no Raft
- ? Garantia: **at-least-once delivery**

### 3. **Failover Automático**

**Problema:** Como lidar com falhas de nós do cluster?

**Solução:** Redirecionamento automático para outro nó.

```csharp
public async Task HandleFailoverAsync()
{
    while (!_cancellationToken.IsCancellationRequested)
    {
        try
        {
            // Tentar conectar ao nó responsável
            var responsibleNode = await GetResponsibleNodeAsync();
            
            if (responsibleNode != _serverStore.NodeTag)
            {
                // Redirecionar para outro nó
                throw new SubscriptionDoesNotBelongToNodeException(
                    $"Subscription '{SubscriptionName}' belongs to node '{responsibleNode}', not '{_serverStore.NodeTag}'");
            }
            
            // Processar normalmente
            await ProcessSubscriptionAsync();
        }
        catch (SubscriptionDoesNotBelongToNodeException)
        {
            // Cliente vai reconectar ao nó correto
            throw;
        }
        catch (Exception ex)
        {
            // Retry com backoff
            var delay = CalculateBackoff(_retryCount++);
            await Task.Delay(delay, _cancellationToken);
        }
    }
}

private string GetResponsibleNode()
{
    using (_serverStore.Engine.ContextPool.AllocateOperationContext(out ClusterOperationContext context))
    using (context.OpenReadTransaction())
    {
        var subscription = GetSubscriptionByName(context, SubscriptionName);
        var topology = _serverStore.Cluster.ReadDatabaseTopology(context, _databaseName);
        
        // Determinar nó responsável
        return topology.WhoseTaskIsIt(
            _serverStore.Engine.CurrentState,
            subscription,
            getLastResponsibleNode: null);
    }
}
```

### 4. **Script-Based Filtering & Projection**

**Problema:** Como permitir filtros complexos sem criar índices?

**Solução:** Executar JavaScript no servidor para filtrar/projetar.

```csharp
public class SubscriptionPatchDocument
{
    private readonly ScriptRunner _scriptRunner;
    
    public SubscriptionPatchDocument(string script, string[] functions)
    {
        _scriptRunner = new ScriptRunner(_database, _database.Configuration, true);
        
        // Adicionar funções auxiliares
        if (functions != null)
        {
            foreach (var function in functions)
                _scriptRunner.AddScript(function);
        }
        
        // Adicionar script principal
        _scriptRunner.AddScript(script);
    }
    
    public bool ApplyFilter(Document doc, DocumentsOperationContext context)
    {
        try
        {
            var result = _scriptRunner.Run(context, context, new
            {
                doc.Id,
                doc.Data,
                Metadata = doc.Data[Constants.Documents.Metadata.Key]
            });
            
            // Resultado deve ser booleano (filter) ou objeto (projection)
            if (result is bool include)
                return include;
            
            if (result is BlittableJsonReaderObject projected)
            {
                // Substituir dados originais por projeção
                doc.Data = projected;
                return true;
            }
            
            return false;
        }
        catch (Exception ex)
        {
            _logger.Error($"Error applying subscription filter for doc {doc.Id}", ex);
            return false;
        }
    }
}
```

**Exemplo de Query:**

```javascript
// Subscription com filtro e projeção
from Orders as o
where o.Total > 1000
select {
    OrderId: o.Id,
    Total: o.Total,
    Customer: load(o.Customer)
}
```

Traduzido para JavaScript:

```javascript
var o = this;
if (o.Total > 1000) {
    return {
        OrderId: o.Id,
        Total: o.Total,
        Customer: loadPath(this, 'Customer')
    };
}
```

### 5. **Connection Pooling e Reuse**

**Problema:** Criar nova conexão TCP para cada reconexão é caro.

**Solução:** Reutilizar conexões existentes quando possível.

```csharp
public class SubscriptionConnectionsState : IDisposable
{
    private readonly ConcurrentSet<SubscriptionConnection> _connections = new();
    private readonly SemaphoreSlim _concurrentConnectionsSemaphore;
    
    public bool TryAddConnection(SubscriptionConnection connection)
    {
        // Limite de conexões concorrentes (padrão: 1)
        if (!_concurrentConnectionsSemaphore.Wait(0))
        {
            throw new SubscriptionInUseException(
                $"Subscription '{SubscriptionName}' is already in use. " +
                $"Maximum concurrent connections: {_maxConcurrentConnections}");
        }
        
        try
        {
            if (_connections.TryAdd(connection))
            {
                connection.OnDispose += () => RemoveConnection(connection);
                return true;
            }
            
            return false;
        }
        catch
        {
            _concurrentConnectionsSemaphore.Release();
            throw;
        }
    }
    
    private void RemoveConnection(SubscriptionConnection connection)
    {
        if (_connections.TryRemove(connection))
        {
            _concurrentConnectionsSemaphore.Release();
        }
    }
}
```

---

## ?? Exemplos de Código Notáveis

### 1. **Cleanup de Subscriptions Idle**

```csharp
internal virtual void CleanupSubscriptions()
{
    var maxTaskLifeTime = _db.Is32Bits 
        ? TimeSpan.FromHours(12) 
        : TimeSpan.FromDays(2);
    
    var oldestPossibleIdleSubscription = SystemTime.UtcNow - maxTaskLifeTime;
    
    foreach (var kvp in _subscriptions)
    {
        if (kvp.Value.IsSubscriptionActive())
            continue; // Ativa, não deletar
        
        var recentConnection = kvp.Value.MostRecentEndedConnection();
        
        if (recentConnection != null && 
            recentConnection.Date < oldestPossibleIdleSubscription)
        {
            // Idle por muito tempo, remover
            if (_subscriptions.TryRemove(kvp.Key, out var subsState))
            {
                subsState.Dispose();
            }
        }
    }
}
```

### 2. **Includes Handling**

```csharp
protected override void GatherIncludesForDocument(
    DatabaseIncludesCommandImpl includeDocuments, 
    Document document)
{
    if (includeDocuments == null)
        return;
    
    // Includes de documentos
    if (_subscription.Includes != null)
    {
        foreach (var includePath in _subscription.Includes)
        {
            var includedDocId = document.Data.GetByPath(includePath);
            if (includedDocId != null)
            {
                includeDocuments.Include(includedDocId.ToString());
            }
        }
    }
    
    // Includes de counters
    if (_subscription.CounterIncludes != null)
    {
        foreach (var counterName in _subscription.CounterIncludes)
        {
            includeDocuments.IncludeCounter(document.Id, counterName);
        }
    }
    
    // Includes de time series
    if (_subscription.TimeSeriesIncludes != null)
    {
        foreach (var ts in _subscription.TimeSeriesIncludes.TimeSeries)
        {
            includeDocuments.IncludeTimeSeries(document.Id, ts.Name, ts.From, ts.To);
        }
    }
}
```

---

## ?? Padrões de Design

### 1. **Observer Pattern**

```csharp
// Subscription é essencialmente um Observer
public class SubscriptionStorage
{
    public event Action<string> OnAddTask;
    public event Action<string> OnRemoveTask;
    public event Action<SubscriptionConnection> OnEndConnection;
    public event Action<string, SubscriptionBatchStatsAggregator> OnEndBatch;
}
```

### 2. **State Pattern**

```csharp
public enum SubscriptionStatus
{
    ConnectionPending,    // Aguardando conexão
    ConnectionActive,     // Conectado e ativo
    BatchSendDocuments,   // Enviando batch
    BatchWaitForAcknowledge, // Aguardando ACK
    Completed,            // Finalizado
    Error                 // Erro
}
```

### 3. **Template Method Pattern**

```csharp
public abstract class SubscriptionConnectionBase
{
    public async Task RunAsync()
    {
        // Template method
        await BeforeConnectionEstablishedAsync();
        await EstablishConnectionAsync();
        await AfterConnectionEstablishedAsync();
        
        while (IsActive)
        {
            await ProcessBatchAsync();
        }
        
        await CleanupAsync();
    }
    
    protected abstract Task ProcessBatchAsync();
    protected abstract Task EstablishConnectionAsync();
}
```

---

## ?? Métricas de Performance

```csharp
public class SubscriptionBatchStatsAggregator
{
    public long NumberOfDocuments { get; set; }
    public long NumberOfDocumentsSkipped { get; set; }
    public long SizeOfDocumentsInBytes { get; set; }
    public long DurationInMs { get; set; }
    
    public double DocsPerSecond => 
        DurationInMs > 0 ? (NumberOfDocuments * 1000.0) / DurationInMs : 0;
    
    public double MBytesPerSecond => 
        DurationInMs > 0 ? (SizeOfDocumentsInBytes / (1024.0 * 1024.0) * 1000.0) / DurationInMs : 0;
}
```

---

## ?? Integração com Outros Módulos

### **Com Raft (Rachis)**
- Progresso persistido via comandos Raft
- Failover coordenado pelo cluster
- `PutSubscriptionCommand`, `AcknowledgeSubscriptionBatchCommand`

### **Com Document Storage**
- `GetDocumentsFrom()` para iterar mudanças
- Change vectors para tracking de progresso

### **Com Replication**
- Subscription funciona em todos os nós
- Failover automático entre nós

---

## ?? Lições Aprendidas

### ? O Que Fazer

1. **Usar batching para eficiência**
   - 4096 docs ou 32 MB por batch
   - Reduz overhead de rede

2. **Rastrear progresso com change vectors**
   - Permite resume após falhas
   - Garantia at-least-once

3. **Implementar retry com backoff**
   - Reconexão automática
   - Backoff exponencial

4. **Usar script filtering quando apropriado**
   - Evita tráfego desnecessário
   - Flexibilidade sem índices

### ?? O Que Evitar

1. **Não confiar apenas em memória**
   - Progresso deve ser persistido
   - Usar Raft para durabilidade

2. **Não criar subscription por documento**
   - Usar filtro na query
   - Batching é essencial

3. **Não ignorar limites de conexão**
   - Default: 1 conexão concorrente
   - Previne duplicação

4. **Não processar batches muito grandes**
   - Limite de 32 MB
   - Previne OOM

---

**? Módulo 9 - Subscriptions System - COMPLETO**

*Próximo módulo recomendado:*
- **Módulo 10**: ETL System (integração com sistemas externos)
