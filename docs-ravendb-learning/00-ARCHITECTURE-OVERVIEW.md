# RavenDB - Visão Geral da Arquitetura

## ??? Arquitetura em Camadas

O RavenDB segue uma arquitetura bem definida em camadas, onde cada projeto tem responsabilidades específicas e depende apenas das camadas inferiores. Esta organização facilita manutenção, testabilidade e reutilização de código.

```
???????????????????????????????????????????????????????????????
?                      RAVEN.STUDIO                           ?
?                   (TypeScript/React)                        ?
?                      Web Interface                          ?
???????????????????????????????????????????????????????????????
                              ?
                              ?
???????????????????????????????????????????????????????????????
?                     RAVEN.SERVER                            ?
?              (Main Database Server - .NET 8)                ?
?   • HTTP API • Query Engine • Clustering • Replication      ?
???????????????????????????????????????????????????????????????
                    ?         ?         ?
                    ?         ?         ?
      ???????????????         ?         ???????????????
      ?                       ?                       ?
      ?                       ?                       ?
????????????          ????????????            ????????????
?  CORAX   ?          ?  VORON   ?            ?  CLIENT  ?
? Search   ?          ? Storage  ?            ? Library  ?
? Engine   ?          ? Engine   ?            ?          ?
????????????          ????????????            ????????????
      ?                       ?                       ?
      ?????????????????????????                       ?
                  ?                                   ?
                  ?                                   ?
          ????????????????                    ????????????
          ?SPARROW.SERVER?                    ? SPARROW  ?
          ?   .NET 8     ?                    ?Multi-TFM ?
          ????????????????                    ????????????
                  ?                                   ?
                  ?????????????????????????????????????
                                  ?
                          ????????????????
                          ?  RAVEN.PAL   ?
                          ?   (C/C++)    ?
                          ?Native Library?
                          ????????????????
```

## ?? Hierarquia de Dependências

### Camada 1: Fundação (Sem Dependências .NET Internas)

#### **Sparrow** (Multi-Target Framework)
```
Targets: net8.0, net7.0, net6.0, netstandard2.1, netstandard2.0
Dependencies: NENHUM projeto interno
Packages: RecyclableMemoryStream, Nito.AsyncEx
```

**Propósito:** Biblioteca de utilitários de baixo nível e alta performance

**Responsabilidades:**
- ? JSON parsing e serialization (Blittable JSON)
- ? Memory management (Arena allocators, pooling)
- ? Estruturas de dados otimizadas (FastList, FastStack, ConcurrentSet)
- ? Compressão (LZ4, Zstd wrappers)
- ? Encoding e parsing de baixo nível
- ? Logging abstraction
- ? Unsafe operations e ponteiros
- ? Low memory notifications

**Técnicas Chave:**
- `Span<T>`, `Memory<T>` para zero-allocation
- `stackalloc` para buffers temporários
- Memory pooling extensivo
- Blittable JSON (formato binário otimizado)

**Por que Multi-Target?**
- Usado pelo Raven.Client (distribuído via NuGet)
- Precisa rodar em diversos ambientes (.NET Framework 4.6.2+, .NET Core, etc.)

---

### Camada 2: Extensões Server-Side

#### **Sparrow.Server** (.NET 8)
```
Targets: net8.0 APENAS
Dependencies: Sparrow
Packages: NLog, Mono.Posix, PerformanceCounter, Numerics.Tensors
```

**Propósito:** Extensões de Sparrow específicas para o servidor

**Responsabilidades:**
- ? Platform-specific code (Windows/Linux/macOS)
- ? Low memory monitoring avançado
- ? Disk I/O statistics
- ? Network utilities (TCP extensions)
- ? Performance metrics collection
- ? String distance algorithms (Levenshtein, Jaro-Winkler)
- ? Compression algorithms (HOPE encoder)
- ? VxSort (sorting vectorizado SIMD)
- ? Tensor operations

**Técnicas Chave:**
- P/Invoke para syscalls POSIX
- SIMD operations (AVX2)
- Hardware intrinsics
- Platform detection

**Por que separado?**
- Código específico de servidor não precisa rodar no client
- Depende de bibliotecas não disponíveis em todos os ambientes
- Permite otimizações específicas de plataforma

---

### Camada 3: Storage & Search Engines

#### **Voron** (.NET 8)
```
Targets: net8.0
Dependencies: Sparrow.Server ? Sparrow
Packages: Numerics.Tensors
```

**Propósito:** Storage engine ACID transacional

**Responsabilidades:**
- ? B+Tree implementation
- ? Memory-mapped files management
- ? ACID transactions
- ? Write-Ahead Logging (WAL)
- ? Page management e allocation
- ? Backup & Recovery
- ? Copy-on-Write (MVCC)
- ? Compression de páginas
- ? HNSW vector search

**Estruturas de Dados:**
- `Tree` - B+Tree principal
- `FixedSizeTree` - Árvores de tamanho fixo
- `CompactTree` - Árvores compactas
- `Table` - Tabelas relacionais
- `PostingList` - Listas de postagem para indexação

**Técnicas Chave:**
- Memory-mapped I/O (mmap)
- Lock-free reads (MVCC)
- Zero-copy operations
- Page-level compression
- FastPFor encoding

**Arquivos Notáveis:**
- `StorageEnvironment.cs` - Entry point
- `Transaction.cs` - Gerenciamento de transações
- `WriteAheadJournal.cs` - WAL implementation
- `AbstractPager.cs` - Memory mapping abstraction

---

#### **Corax** (.NET 8)
```
Targets: net8.0
Dependencies: Voron ? Sparrow.Server ? Sparrow
Packages: Spatial4n, Newtonsoft.Json, Numerics.Tensors
```

**Propósito:** Full-text search engine (substituto moderno do Lucene)

**Responsabilidades:**
- ? Text indexing e tokenization
- ? Query parsing e execution
- ? Scoring (BM25)
- ? Suggestions (fuzzy search)
- ? Spatial queries
- ? Vector search integration
- ? Sorting e aggregations
- ? Phrase matching

**Técnicas Chave:**
- Inverted index usando Voron
- Skip lists
- Bitmap operations
- SIMD string operations
- Entry encoding otimizado

**Arquivos Notáveis:**
- `IndexWriter.cs` - Escrita de índices
- `IndexSearcher.cs` - Busca e queries
- `TermMatch.cs` - Matching de termos
- `SortingMatch.cs` - Ordenação eficiente

---

### Camada 4: Client Library

#### **Raven.Client** (Multi-Target)
```
Targets: net8.0, net7.0, net6.0, netstandard2.1, netstandard2.0
Dependencies: Sparrow (APENAS)
Packages: Newtonsoft.Json, Lambda2Js
```

**Propósito:** Client library para acesso ao RavenDB

**Responsabilidades:**
- ? Session management (Unit of Work)
- ? Change tracking
- ? LINQ provider
- ? HTTP/2 communication
- ? Caching strategies
- ? Subscriptions
- ? Bulk operations
- ? Load balancing

**Padrões de Design:**
- Unit of Work pattern
- Identity Map
- Lazy Loading
- Aggressive Caching
- Repository pattern

**Técnicas Chave:**
- Expression tree transformation (LINQ ? RQL)
- HTTP connection pooling
- JSON serialization otimizada
- Retry policies

**Por que depende APENAS de Sparrow?**
- Minimiza dependências do cliente
- Facilita distribuição via NuGet
- Sparrow fornece JSON e utilities básicos

---

### Camada 5: Server Application

#### **Raven.Server** (.NET 8)
```
Targets: net8.0
Dependencies: Corax, Voron, Sparrow.Server, Raven.Client
Packages: 100+ packages (AWS, Azure, Kafka, AI models, etc.)
```

**Propósito:** Aplicação servidor principal

**Responsabilidades:**
- ? HTTP API (ASP.NET Core)
- ? Query engine (RQL)
- ? Document processing
- ? Indexing orchestration
- ? Clustering (Raft consensus)
- ? Replication
- ? ETL (Extract, Transform, Load)
- ? Backups
- ? Migrations
- ? Monitoring
- ? AI integrations

**Módulos Principais:**
```
Documents/
??? Commands/          # HTTP endpoints
??? Queries/           # Query execution
??? Indexes/           # Index management
??? Handlers/          # Request handlers
??? Replication/       # Data replication
??? ETL/              # ETL processes

Server/
??? Rachis/           # Raft consensus
??? ServerWide/       # Cluster management
??? NotificationCenter/

Storage/
??? DocumentsStorage/  # Document persistence
```

**Técnicas Chave:**
- Request pipelining
- Async I/O
- Pooling de recursos
- Transaction batching
- Streaming de resultados

---

### Camada 6: User Interface

#### **Raven.Studio** (TypeScript/React)
```
Technology: Node.js 20+, TypeScript, React, Webpack
Build Time: ~10 minutos
```

**Propósito:** Interface web administrativa

**Responsabilidades:**
- ? Database management UI
- ? Query editor
- ? Index management
- ? Monitoring dashboards
- ? Backup/restore UI
- ? Cluster management

---

## ?? Padrões de Comunicação

### 1. Server ? Storage (Voron)

```csharp
// Raven.Server cria transações no Voron
using (var tx = _environment.WriteTransaction())
{
    var tree = tx.CreateTree("Documents");
    tree.Add("users/1", documentData);
    tx.Commit();
}
```

**Padrão:** Direct method calls (mesma AppDomain)

---

### 2. Server ? Search (Corax)

```csharp
// Raven.Server usa Corax para indexação
using (var indexWriter = new IndexWriter(...))
{
    indexWriter.Index(documentId, fields);
    indexWriter.Commit();
}

// E para busca
var results = indexSearcher.TermQuery("Name", "John");
```

**Padrão:** Direct method calls

---

### 3. Client ? Server

```csharp
// HTTP/2 over TLS
POST /databases/Northwind/docs HTTP/2
Content-Type: application/json

{ "Name": "John", "Age": 30 }
```

**Padrão:** RESTful HTTP API (JSON over HTTP/2)

**Otimizações:**
- Connection pooling
- Keep-alive connections
- Request batching
- Response compression

---

### 4. Server Clustering (Raft)

```
Node1 ??[Raft]??> Node2
  ?               ?
  ??[Raft]???> Node3
```

**Padrão:** Raft consensus protocol
- Leader election
- Log replication
- State machine replication

---

## ?? Fluxo de Dados Típico

### Escrita de Documento

```
Client
  ? 1. HTTP POST
  ?
Raven.Server (DocumentsStorage)
  ? 2. Validate & Transform
  ?
Voron (Transaction)
  ? 3. Write to WAL
  ? 4. Update B+Tree
  ? 5. Commit
  ?
Disk (Memory-mapped file)
  
  ? (Async background)
  ?
Corax (Indexing)
  ? 6. Extract terms
  ? 7. Update inverted index
  ?
Voron (Index storage)
```

### Leitura com Query

```
Client
  ? 1. LINQ query
  ?
Raven.Client
  ? 2. Transform to RQL
  ?
Raven.Server (QueryEngine)
  ? 3. Parse & optimize
  ? 4. Select index
  ?
Corax (IndexSearcher)
  ? 5. Execute query
  ? 6. Score results
  ?
Voron (Read documents)
  ? 7. Load from B+Tree
  ?
Raven.Server
  ? 8. Project & transform
  ?
Client (Materialized objects)
```

---

## ?? Bibliotecas Nativas (Raven.PAL)

### librvnpal (C/C++)
**Propósito:** Platform Abstraction Layer

**Funcionalidades:**
- Crypto operations (libsodium wrapper)
- File system operations otimizadas
- Memory management low-level
- Threading primitives

**Plataformas:**
- `librvnpal.win.x64.dll` (Windows x64)
- `librvnpal.linux.x64.so` (Linux x64)
- `librvnpal.mac.x64.dylib` (macOS x64)
- `librvnpal.arm.64.so` (ARM64)
- etc.

### libzstd
**Propósito:** Zstandard compression library

**Uso:**
- Backup compression
- Network compression
- Page compression

---

## ?? Métricas de Complexidade

| Projeto | Linhas de Código | Arquivos | Complexidade |
|---------|-----------------|----------|--------------|
| Sparrow | ~50k | ~150 | Médio |
| Sparrow.Server | ~30k | ~150 | Médio |
| Voron | ~80k | ~250 | Alto |
| Corax | ~40k | ~140 | Alto |
| Raven.Client | ~100k | ~500 | Alto |
| Raven.Server | ~300k | ~2000 | Muito Alto |

---

## ?? Decisões Arquiteturais Importantes

### 1. Por que Sparrow é Multi-Target?

**Decisão:** Suportar net8.0 até netstandard2.0

**Razão:**
- Usado pelo Raven.Client (distribuído publicamente)
- Clientes podem estar em .NET Framework 4.6.2+
- Minimiza breaking changes

**Trade-off:**
- Não pode usar features modernas do .NET em alguns targets
- Mais complexidade de build
- Mais testes necessários

### 2. Por que Sparrow.Server é separado?

**Decisão:** Split server-specific utilities

**Razão:**
- Código específico de servidor não precisa rodar no cliente
- Permite usar .NET 8 apenas (features modernas)
- Depende de bibliotecas não disponíveis no cliente (NLog, Mono.Posix)

**Benefício:**
- Raven.Client fica menor
- Menos dependências transitivas para clientes

### 3. Por que Voron não usa Async?

**Decisão:** Storage engine é síncrono

**Razão:**
- Memory-mapped I/O é inerentemente síncrono
- Overhead de async/await desnecessário
- Melhor controle sobre transações

**Camada async:**
- Raven.Server adiciona async wrappers quando necessário

### 4. Por que custom JSON (Blittable)?

**Decisão:** Formato JSON binário customizado

**Razão:**
- Zero-copy deserialization
- Acesso direto a campos sem parsing
- Menor uso de memória
- Mais rápido que JSON.NET para workloads típicos

**Trade-off:**
- Mais código para manter
- Curva de aprendizado

### 5. Por que Corax em vez de Lucene?

**Decisão:** Criar search engine próprio

**Razão:**
- Integração nativa com Voron
- Melhor performance para workloads do RavenDB
- Controle total sobre otimizações
- Menos overhead de marshaling

**Trade-off:**
- Mais código para manter
- Precisa implementar features do zero

---

## ?? Pontos de Extensibilidade

### 1. Custom Analyzers (Corax)
```csharp
public class MyAnalyzer : Analyzer
{
    public override IEnumerable<Token> Tokenize(string text)
    {
        // Custom tokenization
    }
}
```

### 2. Custom Indexes
```csharp
public class MyIndex : AbstractIndexCreationTask<Document>
{
    public MyIndex()
    {
        Map = docs => from doc in docs
                      select new { doc.Name };
    }
}
```

### 3. ETL Transformations
```javascript
// JavaScript ETL script
this.Name = this.FirstName + ' ' + this.LastName;
```

### 4. Subscriptions
```csharp
var subscription = await store.Subscriptions
    .CreateAsync<Order>(x => x.Total > 100);
```

---

## ?? Performance-Critical Paths

### Hot Paths Identificados:

1. **Document Write Path**
   - JsonOperationContext (pooling)
   - Blittable JSON parsing
   - Voron transaction commit

2. **Query Execution**
   - Index selection
   - Corax query execution
   - Result materialization

3. **Indexing**
   - Document parsing
   - Term extraction
   - Batch writing to Voron

4. **Replication**
   - Change vector comparison
   - Conflict resolution
   - Network I/O

---

## ?? Principais Padrões de Performance

### 1. Pooling Universal
- `JsonOperationContext` pool
- `ArrayPool<T>`
- `RecyclableMemoryStream`
- Custom object pools

### 2. Zero-Allocation
- `Span<T>` e `Memory<T>`
- `stackalloc`
- Struct enumerators
- Blittable JSON

### 3. Batching
- Transaction batching
- Network request batching
- Index write batching

### 4. Lock-Free
- MVCC em Voron
- Copy-on-Write
- Atomic operations

### 5. Async Otimizado
- `ValueTask<T>` em hot paths
- ConfigureAwait(false)
- Custom SynchronizationContext

---

## ?? Próximos Passos

Agora que você entende a arquitetura geral, escolha:

1. **[Sparrow](01-Sparrow.md)** - Fundação de performance
2. **[Voron](02-Voron.md)** - Storage engine
3. **[Sparrow.Server](03-Sparrow-Server.md)** - Server utilities
4. **[Corax](04-Corax.md)** - Search engine
5. **[Raven.Server](05-Raven-Server.md)** - Main application
6. **[Raven.Client](06-Raven-Client.md)** - Client patterns

### ?? Deep Dive: Raven.Server Modules

Para uma análise **extremamente detalhada** do Raven.Server, veja:

**[?? Raven.Server Deep Dive Modules](raven-server-modules/)**

#### ? Módulos Disponíveis:
- **[Módulo 2: Transaction Merging System](raven-server-modules/MODULE-02-Transaction-Merging.md)** ?
  - O **coração da performance** do RavenDB
  - Async Commit Overlap (40-50% ganho)
  - 5 técnicas avançadas de performance
  - 5 padrões de design detalhados
  - Complexidade: ????? MUITO ALTA

#### ?? Planejamento Completo:
- **[RAVEN-SERVER-DEEP-DIVE-PLAN.md](RAVEN-SERVER-DEEP-DIVE-PLAN.md)** - Plano de 15 módulos
- **Progresso:** 1/15 módulos (6.7%)
- **Total:** ~45-60 horas de análise profunda

#### Próximos Módulos (em planejamento):
1. HTTP Request Pipeline & Routing
3. DocumentDatabase Lifecycle
4. Document Storage & CRUD
5. Indexing Subsystem ?
6. Query Execution Engine ?
7. Replication Engine
8. Clustering & Raft ?

? = Crítico para performance

---

Ou vá direto para análises temáticas:
- **[Performance Patterns](09-Performance-Patterns.md)** - Padrões cruzados
- **[Memory Management](10-Memory-Management.md)** - Estratégias de memória
- **[Concurrency Patterns](11-Concurrency-Patterns.md)** - Padrões de concorrência

---

**Dica:** Comece com **Sparrow** - é a base de tudo e contém as técnicas mais fundamentais que são reutilizadas em todos os outros projetos. Depois, leia o **Módulo 2 (Transaction Merging)** para entender o coração da performance do servidor.
