# Raven.Server - Módulo 10: ETL (Extract-Transform-Load)

## ?? Visão Geral

O **ETL System** permite **integração em tempo real** com sistemas externos, transformando e enviando documentos do RavenDB para destinos como **outros bancos RavenDB**, **bancos SQL**, **OLAP**, **Elastic Search**, **Message Queues** (Kafka, RabbitMQ), e muito mais.

### Responsabilidades Principais

- **Extract**: Ler mudanças de documentos
- **Transform**: Aplicar scripts JavaScript para transformar dados
- **Load**: Enviar para destino externo
- **Monitoring**: Rastrear progresso e erros
- **Failover**: Reconexão automática e retry

### Providers Suportados

```
ETL Providers:
??? RavenEtl          # Outro cluster RavenDB
??? SqlEtl            # SQL Server, MySQL, PostgreSQL, etc
??? OlapEtl           # Parquet files (S3, Azure, Local)
??? ElasticSearchEtl  # ElasticSearch / OpenSearch
??? QueueEtl          # Message brokers
?   ??? KafkaEtl
?   ??? RabbitMqEtl
?   ??? AmazonSqsEtl
?   ??? AzureQueueStorageEtl
??? SnowflakeEtl      # Snowflake data warehouse
??? EmbeddingsGenerationEtl  # AI embeddings
??? GenAiEtl          # GenAI integrations
```

---

## ??? Arquitetura Interna

### 1. **EtlLoader - Orquestrador Central**

```csharp
public sealed class EtlLoader : IDisposable, ITombstoneAware
{
    private EtlProcess[] _processes = new EtlProcess[0];
    private readonly DocumentDatabase _database;
    private readonly ServerStore _serverStore;
    
    // Listas de destinos
    public List<RavenEtlConfiguration> RavenDestinations;
    public List<SqlEtlConfiguration> SqlDestinations;
    public List<OlapEtlConfiguration> OlapDestinations;
    public List<ElasticSearchEtlConfiguration> ElasticSearchDestinations;
    public List<QueueEtlConfiguration> QueueDestinations;
    public List<SnowflakeEtlConfiguration> SnowflakeDestinations;
    // ... e mais
    
    // Events
    public event Action<(string ConfigurationName, string TransformationName, EtlProcessStatistics Statistics)> BatchCompleted;
    public event Action<EtlProcess> ProcessAdded;
    public event Action<EtlProcess> ProcessRemoved;
    
    public void Initialize(DatabaseRecord record)
    {
        LoadProcesses(record, 
            record.RavenEtls, 
            record.SqlEtls, 
            record.OlapEtls,
            // ... todos os providers
        );
    }
}
```

### 2. **EtlProcess - Classe Base Abstrata**

```csharp
public abstract class EtlProcess : IDisposable
{
    protected readonly DocumentDatabase _database;
    protected readonly ServerStore _serverStore;
    protected readonly string ConfigurationName;
    protected readonly string TransformationName;
    
    // Estado do ETL (progresso)
    public EtlProcessState State { get; private set; }
    
    // Estatísticas
    public EtlProcessStatistics Statistics { get; }
    
    // Lifecycle
    public abstract void Start(string reason);
    public abstract void Stop(string reason);
    
    // Processamento
    protected abstract IEnumerable<ToEtlItem> ConvertDocsAndTombstones(
        DocumentsOperationContext context,
        IEnumerable<DocumentTombstone> tombstones,
        string collection);
    
    protected abstract void LoadInternal(
        IEnumerable<ToEtlItem> items,
        JsonOperationContext context);
}
```

### 3. **ETL Configuration**

```csharp
public abstract class EtlConfiguration<TConnectionString> 
    where TConnectionString : ConnectionString
{
    public string Name { get; set; }
    public string ConnectionStringName { get; set; }
    public List<Transformation> Transforms { get; set; }
    public bool Disabled { get; set; }
    public bool AllowEtlOnNonEncryptedChannel { get; set; }
    public TConnectionString Connection { get; set; }
    
    public abstract EtlType EtlType { get; }
    
    public bool Validate(out List<string> errors)
    {
        errors = new List<string>();
        
        if (string.IsNullOrEmpty(Name))
            errors.Add("Name cannot be empty");
        
        if (string.IsNullOrEmpty(ConnectionStringName))
            errors.Add("Connection string name cannot be empty");
        
        if (Transforms == null || Transforms.Count == 0)
            errors.Add("At least one transformation must be defined");
        
        return errors.Count == 0;
    }
}

public class Transformation
{
    public string Name { get; set; }
    public List<string> Collections { get; set; }
    public bool ApplyToAllDocuments { get; set; }
    public string Script { get; set; }
}
```

---

## ?? Fluxo de ETL

### **End-to-End Flow - Exemplo com SQL ETL**

```
???????????????????????????????????????????????????????????????
?           1. CONFIGURAÇÃO DO ETL (via Studio/API)           ?
???????????????????????????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  ETL Configuration Criada             ?
        ?  {                                    ?
        ?    Name: "Orders to SQL",             ?
        ?    ConnectionString: "SQL-Prod",      ?
        ?    Transforms: [{                     ?
        ?      Collections: ["Orders"],         ?
        ?      Script: "loadToOrders({          ?
        ?        OrderId: this.Id,              ?
        ?        Total: this.Total              ?
        ?      })"                              ?
        ?    }]                                 ?
        ?  }                                    ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  2. Comando Raft                      ?
        ?     PutDatabaseRecordCommand          ?
        ?     - Replicado para cluster          ?
        ?     - Persistido                      ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  3. EtlLoader.HandleDatabaseRecordChange()
        ?     - Detecta nova configuração       ?
        ?     - Determina nó responsável        ?
        ?     - Cria SqlEtl process             ?
        ????????????????????????????????????????
                           ?
                           ?
??????????????????????????????????????????????????????????????
?            4. ETL PROCESS RUNNING                          ?
??????????????????????????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  5. Subscribe to Changes              ?
        ?     database.Changes.OnDocumentChange ?
        ?     - Notificações de mudanças        ?
        ????????????????????????????????????????
                           ?
                           ?
??????????????????????????????????????????????????????????????
?                 LOOP DE PROCESSAMENTO                      ?
??????????????????????????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  6. Extract - Ler Documentos          ?
        ?     - Query desde lastCV              ?
        ?     - GetDocumentsFrom()              ?
        ?     - Batch de até 1024 docs          ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  7. Transform - Executar Script       ?
        ?     var order = this;                 ?
        ?     loadToOrders({                    ?
        ?       OrderId: order.Id,              ?
        ?       Total: order.Total              ?
        ?     });                               ?
        ?     - Jint JavaScript engine          ?
        ?     - Output: SQL commands            ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  8. Load - Enviar para Destino        ?
        ?     SQL:                              ?
        ?     - Gerar INSERT/UPDATE/DELETE      ?
        ?     - Executar em transação           ?
        ?     - Batch de comandos               ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  9. Commit - Atualizar Progresso      ?
        ?     - Salvar lastCV no Raft           ?
        ?     - EtlProcessState.ChangeVector    ?
        ?     - Permitir tombstone cleanup      ?
        ????????????????????????????????????????
                           ?
                           ?
        ????????????????????????????????????????
        ?  10. Estatísticas                     ?
        ?     - Docs processados                ?
        ?     - Erros                           ?
        ?     - Throughput                      ?
        ?     - Notificar BatchCompleted        ?
        ????????????????????????????????????????
                           ?
                           ?
                           ??????????? Loop continua
                                     ?
                                     ?
```

---

## ?? Técnicas de Performance

### 1. **Batching de Documentos**

**Problema:** Processar cada documento individualmente é ineficiente.

**Solução:** Agrupar documentos em batches.

```csharp
public abstract class EtlProcess
{
    protected const int MaxBatchSize = 1024;
    protected const int MaxBatchSizeInBytes = 16 * 1024 * 1024; // 16 MB
    
    protected IEnumerable<Document> GetDocumentsBatch(
        DocumentsOperationContext context,
        string collection,
        string fromChangeVector)
    {
        var batch = new List<Document>();
        var totalSize = 0;
        
        var docs = _database.DocumentsStorage.GetDocumentsFrom(
            context,
            collection,
            fromChangeVector,
            start: 0,
            pageSize: MaxBatchSize);
        
        foreach (var doc in docs)
        {
            batch.Add(doc);
            totalSize += doc.Data.Size;
            
            if (batch.Count >= MaxBatchSize || 
                totalSize >= MaxBatchSizeInBytes)
            {
                yield return batch;
                batch = new List<Document>();
                totalSize = 0;
            }
        }
        
        if (batch.Count > 0)
            yield return batch;
    }
}
```

**Ganho:** Reduz overhead de rede e transações SQL em 10-100x.

### 2. **Script Caching (Jint Compilation)**

**Problema:** Compilar scripts JavaScript a cada execução é caro.

**Solução:** Cachear scripts compilados.

```csharp
public class EtlScriptRunner
{
    private readonly ConcurrentDictionary<string, ScriptRunner.SingleRun> _scriptCache 
        = new ConcurrentDictionary<string, ScriptRunner.SingleRun>();
    
    public object RunScript(string script, Document doc, JsonOperationContext context)
    {
        // Obter script compilado do cache
        var compiledScript = _scriptCache.GetOrAdd(script, s =>
        {
            // Compilar uma única vez
            var runner = new ScriptRunner(_database, _database.Configuration, forAdHocQuery: false);
            runner.AddScript(s);
            return runner.Compile();
        });
        
        // Executar com documento atual
        return compiledScript.Run(context, context, new
        {
            doc.Id,
            doc.Data,
            Metadata = doc.Data[Constants.Documents.Metadata.Key]
        });
    }
}
```

**Ganho:** Primeira execução ~100ms, subsequentes ~1ms. **100x mais rápido**.

### 3. **Connection Pooling para SQL**

**Problema:** Criar nova conexão SQL para cada batch é lento.

**Solução:** Reutilizar conexões via pooling.

```csharp
public class SqlEtl : EtlProcess
{
    private readonly ConnectionStringSettings _connectionString;
    
    protected override void LoadInternal(
        IEnumerable<ToEtlItem> items,
        JsonOperationContext context)
    {
        // ADO.NET faz pooling automaticamente
        using (var connection = new SqlConnection(_connectionString.ConnectionString))
        {
            connection.Open();
            
            using (var transaction = connection.BeginTransaction())
            {
                foreach (var item in items)
                {
                    var sqlCommand = GenerateSqlCommand(item);
                    
                    using (var cmd = new SqlCommand(sqlCommand, connection, transaction))
                    {
                        cmd.ExecuteNonQuery();
                    }
                }
                
                transaction.Commit();
            }
        }
        // Conexão retorna ao pool ao invés de ser fechada
    }
}
```

**Ganho:** Latência de conexão reduz de 50ms para <1ms.

### 4. **Fallback Script on Error**

**Problema:** Script com erro pode parar o ETL completamente.

**Solução:** Script alternativo para lidar com erros.

```csharp
public class Transformation
{
    public string Script { get; set; }
    public string FallbackScript { get; set; }
    
    public object Execute(Document doc, JsonOperationContext context)
    {
        try
        {
            return RunScript(Script, doc, context);
        }
        catch (Exception ex)
        {
            if (string.IsNullOrEmpty(FallbackScript))
                throw;
            
            try
            {
                return RunScript(FallbackScript, doc, context);
            }
            catch (Exception fallbackEx)
            {
                throw new AggregateException(
                    "Both main and fallback scripts failed", 
                    ex, 
                    fallbackEx);
            }
        }
    }
}
```

**Resultado:** ETL continua funcionando mesmo com erros ocasionais.

### 5. **Tombstone Coordination**

**Problema:** Deletar tombstones antes de ETL processá-los causa perda de dados.

**Solução:** Coordenar cleanup de tombstones com progresso do ETL.

```csharp
public interface ITombstoneAware
{
    Dictionary<string, long> GetLastProcessedTombstonesPerCollection(
        TombstoneType tombstoneType);
}

public sealed class EtlLoader : ITombstoneAware
{
    public Dictionary<string, long> GetLastProcessedTombstonesPerCollection(
        ITombstoneAware.TombstoneType tombstoneType,
        Dictionary<string, LastTombstoneInfo> lastProcessedTombstonesInfo = null)
    {
        var lastProcessedTombstones = new Dictionary<string, long>();
        
        // Para cada ETL, obter último tombstone processado
        foreach (var config in _databaseRecord.RavenEtls)
        {
            foreach (var transform in config.Transforms)
            {
                var state = EtlProcess.GetProcessState(
                    _database, 
                    config.Name, 
                    transform.Name);
                
                var etag = ChangeVectorUtils.GetEtagById(
                    state.ChangeVector, 
                    _database.DbBase64Id);
                
                // Marcar tombstones que podem ser deletados
                foreach (var collection in transform.Collections)
                {
                    if (!lastProcessedTombstones.TryGetValue(collection, out var currentEtag) ||
                        etag < currentEtag)
                    {
                        lastProcessedTombstones[collection] = etag;
                    }
                }
            }
        }
        
        return lastProcessedTombstones;
    }
}
```

**Resultado:** Tombstones só são deletados após **TODOS** os ETLs processarem.

---

## ?? Exemplos de Código Notáveis

### 1. **Dynamic ETL Process Creation**

```csharp
private IEnumerable<EtlProcess> GetRelevantProcesses<T, TConnectionString>(
    List<T> configurations, 
    HashSet<string> uniqueNames) 
    where T : EtlConfiguration<TConnectionString> 
    where TConnectionString : ConnectionString
{
    foreach (var config in configurations)
    {
        // 1. Validar configuração
        if (ValidateConfiguration(config, uniqueNames) == false)
            continue;
        
        // 2. Determinar nó responsável
        var processState = GetProcessState(config.Transforms, _database, config.Name);
        var whoseTaskIsIt = OngoingTasksUtils.WhoseTaskIsIt(
            _serverStore, 
            _databaseRecord.Topology, 
            config, 
            processState, 
            _database.NotificationCenter);
        
        if (whoseTaskIsIt != _serverStore.NodeTag)
            continue; // Outro nó é responsável
        
        // 3. Criar processo específico para cada transformação
        foreach (var transform in config.Transforms)
        {
            EtlProcess process = null;
            
            switch (config.EtlType)
            {
                case EtlType.Sql:
                    process = new SqlEtl(transform, config as SqlEtlConfiguration, _database, _serverStore);
                    break;
                case EtlType.Raven:
                    process = new RavenEtl(transform, config as RavenEtlConfiguration, _database, _serverStore);
                    break;
                case EtlType.Olap:
                    process = new OlapEtl(transform, config as OlapEtlConfiguration, _database, _serverStore);
                    break;
                case EtlType.Queue:
                    process = QueueEtl<QueueItem>.CreateInstance(transform, config as QueueEtlConfiguration, _database, _serverStore);
                    break;
                // ... outros tipos
            }
            
            yield return process;
        }
    }
}
```

### 2. **Change Detection and Subscription**

```csharp
private void HandleChangesSubscriptions()
{
    if (_processes.Length > 0)
    {
        // Subscribe to document changes
        if (_isSubscribedToDocumentChanges == false)
        {
            _database.Changes.OnDocumentChange += OnDocumentChange;
            _isSubscribedToDocumentChanges = true;
        }
        
        // Subscribe to counter changes se algum ETL precisa
        var needToWatchCounters = _processes.Any(x => x.ShouldTrackCounters());
        if (needToWatchCounters)
        {
            if (_isSubscribedToCounterChanges == false)
            {
                _database.Changes.OnCounterChange += OnCounterChange;
                _isSubscribedToCounterChanges = true;
            }
        }
        else
        {
            _database.Changes.OnCounterChange -= OnCounterChange;
            _isSubscribedToCounterChanges = false;
        }
        
        // Subscribe to time series changes se algum ETL precisa
        var needToWatchTimeSeries = _processes.Any(x => x.ShouldTrackTimeSeries());
        if (needToWatchTimeSeries)
        {
            if (_isSubscribedToTimeSeriesChanges == false)
            {
                _database.Changes.OnTimeSeriesChange += OnTimeSeriesChange;
                _isSubscribedToTimeSeriesChanges = true;
            }
        }
    }
    else
    {
        // Nenhum processo, unsubscribe de tudo
        _database.Changes.OnDocumentChange -= OnDocumentChange;
        _database.Changes.OnCounterChange -= OnCounterChange;
        _database.Changes.OnTimeSeriesChange -= OnTimeSeriesChange;
    }
}

private void OnDocumentChange(DocumentChange change)
{
    // Notificar todos os processos
    var processes = _processes;
    for (var i = 0; i < processes.Length; i++)
    {
        try
        {
            processes[i].NotifyAboutWork(change);
        }
        catch (ObjectDisposedException)
        {
            // Process foi disposto, ignorar
        }
    }
}
```

### 3. **Configuration Comparison and Hot Reload**

```csharp
public void HandleDatabaseRecordChange(DatabaseRecord record)
{
    var toRemove = _processes.GroupBy(x => x.ConfigurationName)
                             .ToDictionary(x => x.Key, x => x.ToList());
    
    // Comparar configurações antigas vs novas
    foreach (var processesPerConfig in _processes.GroupBy(x => x.ConfigurationName))
    {
        var process = processesPerConfig.First();
        
        if (process is SqlEtl sqlEtl)
        {
            SqlEtlConfiguration existing = null;
            
            foreach (var config in mySqlEtl)
            {
                // Comparar configurações
                var diff = sqlEtl.Configuration.Compare(
                    config, 
                    record.SqlConnectionStrings);
                
                if (diff == EtlConfigurationCompareDifferences.None)
                {
                    existing = config;
                    break; // Nenhuma mudança
                }
            }
            
            if (existing != null)
            {
                // Configuração não mudou, manter processo
                toRemove.Remove(processesPerConfig.Key);
                mySqlEtl.Remove(existing);
            }
        }
    }
    
    // Parar processos removidos/alterados
    foreach (var processList in toRemove.Values)
    {
        foreach (var process in processList)
        {
            process.Stop("Configuration changed");
            process.Dispose();
        }
    }
    
    // Criar processos novos/alterados
    LoadProcesses(record, myRavenEtl, mySqlEtl, ...);
}
```

---

## ?? Padrões de Design

### 1. **Strategy Pattern - ETL Providers**

```csharp
// Interface comum
public abstract class EtlProcess
{
    protected abstract void LoadInternal(IEnumerable<ToEtlItem> items, JsonOperationContext context);
}

// Estratégias específicas
public class SqlEtl : EtlProcess
{
    protected override void LoadInternal(...)
    {
        // Gerar e executar SQL
    }
}

public class RavenEtl : EtlProcess
{
    protected override void LoadInternal(...)
    {
        // Enviar via HTTP para outro RavenDB
    }
}

public class KafkaEtl : QueueEtl<QueueItem>
{
    protected override void LoadInternal(...)
    {
        // Enviar mensagens para Kafka
    }
}
```

### 2. **Template Method Pattern**

```csharp
public abstract class EtlProcess
{
    public void Run()
    {
        while (!_cancellationToken.IsCancellationRequested)
        {
            // Template method
            BeforeProcessing();
            
            var items = Extract();      // Hook
            items = Transform(items);   // Hook
            Load(items);                // Hook
            
            AfterProcessing();
        }
    }
    
    protected abstract IEnumerable<ToEtlItem> Extract();
    protected abstract IEnumerable<ToEtlItem> Transform(IEnumerable<ToEtlItem> items);
    protected abstract void Load(IEnumerable<ToEtlItem> items);
}
```

### 3. **Observer Pattern**

```csharp
// ETL observa mudanças no database
public sealed class EtlLoader
{
    private void OnDocumentChange(DocumentChange change)
    {
        // Notificar todos os processos
        foreach (var process in _processes)
        {
            process.NotifyAboutWork(change);
        }
    }
}
```

---

## ?? Métricas de Performance

```csharp
public class EtlProcessStatistics
{
    public long LastProcessedEtag { get; set; }
    public string LastChangeVector { get; set; }
    public DateTime LastBatchTime { get; set; }
    public long NumberOfExtractedItems { get; set; }
    public long NumberOfTransformedItems { get; set; }
    public long NumberOfLoadedItems { get; set; }
    public long NumberOfErroredItems { get; set; }
    public TimeSpan Duration { get; set; }
    
    public double ItemsPerSecond => 
        Duration.TotalSeconds > 0 
            ? NumberOfLoadedItems / Duration.TotalSeconds 
            : 0;
}
```

---

## ?? Integração com Outros Módulos

### **Com Raft (Rachis)**
- Configuração via comandos Raft
- Estado persistido no cluster
- Failover coordenado

### **Com Document Storage**
- `GetDocumentsFrom()` para extrair mudanças
- Change vectors para tracking

### **Com Subscriptions**
- Ambos usam change tracking
- Patterns similares de batching

---

## ?? Lições Aprendidas

### ? O Que Fazer

1. **Usar batching agressivamente**
   - 1024 docs ou 16 MB por batch
   - Reduz overhead massivamente

2. **Cachear scripts compilados**
   - Jint compilation é caro
   - 100x ganho com cache

3. **Coordenar tombstone cleanup**
   - Aguardar TODOS os ETLs
   - Previne perda de deletes

4. **Implementar fallback scripts**
   - Resiliência em erros
   - ETL continua funcionando

### ?? O Que Evitar

1. **Não processar documento por documento**
   - Sempre usar batches
   - Overhead de rede/transações é alto

2. **Não ignorar erros de transformação**
   - Log detalhado
   - Fallback script quando possível

3. **Não criar conexões SQL repetidamente**
   - Usar connection pooling
   - ADO.NET faz automaticamente

4. **Não confiar apenas em memória**
   - Persistir progresso no Raft
   - Change vector é crítico

---

**? Módulo 10 - ETL System - COMPLETO**

*Próximo módulo recomendado:*
- **Módulo 11**: Periodic Backup System (disaster recovery)
