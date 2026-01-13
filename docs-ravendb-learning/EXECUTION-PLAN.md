# Plano de Análise Individual por Projeto

Este documento contém prompts específicos para análise profunda de cada projeto do RavenDB.

## ?? Como Usar

Copie e cole cada prompt individualmente para obter análise detalhada de cada componente.

---

## Prompt 1: Sparrow (Fundação de Performance)

```
Analise o projeto Sparrow (src/Sparrow) do RavenDB em profundidade, focando em:

1. **Estruturas de Dados Otimizadas**
   - Identifique collections customizadas
   - Analise implementações de pooling
   - Documente uso de Span<T>, Memory<T>, ArrayPool

2. **Gerenciamento de Memória**
   - Code unsafe e ponteiros
   - Stack allocation (stackalloc)
   - Memory alignment
   - Técnicas de zero-allocation

3. **Utilitários de Baixo Nível**
   - Operações de bits
   - SIMD e intrinsics
   - Interop e P/Invoke

4. **Padrões de Performance**
   - Inlining strategies
   - Branch prediction
   - Cache-friendly code

Crie o arquivo docs-ravendb-learning/01-Sparrow.md com exemplos de código, explicações detalhadas e lições aprendidas.
```

---

## Prompt 2: Voron (Storage Engine)

```
Analise o projeto Voron (src/Voron) do RavenDB em profundidade, focando em:

1. **Storage Engine Architecture**
   - B+Tree implementation
   - Page management
   - Transaction handling
   - ACID guarantees

2. **Memory-Mapped Files**
   - Memory mapping strategies
   - Page cache management
   - Write-ahead logging (WAL)

3. **Performance Techniques**
   - Lock-free reads
   - Copy-on-write
   - Batch operations
   - Compression

4. **Data Structures**
   - Tree structures
   - Free space management
   - Page allocation

Crie o arquivo docs-ravendb-learning/02-Voron.md com análise detalhada da implementação e técnicas de performance.
```

---

## Prompt 3: Sparrow.Server (Server-Side Extensions)

```
Analise o projeto Sparrow.Server (src/Sparrow.Server) do RavenDB em profundidade, focando em:

1. **Network Optimization**
   - Socket management
   - Buffer pooling
   - Zero-copy techniques
   - Protocol optimization

2. **Threading e Async**
   - Custom thread pools
   - Async/await patterns
   - Task scheduling
   - Synchronization primitives

3. **Resource Management**
   - Connection pooling
   - Resource cleanup
   - Disposal patterns
   - Memory pressure handling

4. **Performance Monitoring**
   - Metrics collection
   - Performance counters
   - Diagnostics

Crie o arquivo docs-ravendb-learning/03-Sparrow-Server.md com análise detalhada.
```

---

## Prompt 4: Corax (Search Engine)

```
Analise o projeto Corax (src/Corax) do RavenDB em profundidade, focando em:

1. **Indexing Architecture**
   - Full-text indexing
   - Term dictionaries
   - Posting lists
   - Index compression

2. **Query Processing**
   - Query parsing
   - Query optimization
   - Scoring algorithms
   - Result ranking

3. **Performance Techniques**
   - Inverted index structures
   - Skip lists
   - Bitmap operations
   - Memory-efficient structures

4. **Text Processing**
   - Tokenization
   - Analyzers
   - String interning
   - Unicode handling

Crie o arquivo docs-ravendb-learning/04-Corax.md com análise detalhada.
```

---

## Prompt 5: Raven.Server (Main Server)

```
Analise o projeto Raven.Server (src/Raven.Server) do RavenDB em profundidade, focando em:

1. **Request Pipeline**
   - HTTP handling
   - Request routing
   - Middleware patterns
   - Response streaming

2. **Document Processing**
   - Document storage
   - Change tracking
   - Patching operations
   - Serialization/deserialization

3. **Query Execution**
   - Query planning
   - Index selection
   - Query optimization
   - Result streaming

4. **Clustering e Replicação**
   - Raft consensus
   - Replication protocol
   - Conflict resolution
   - Distributed transactions

5. **Concurrency Control**
   - Document-level locking
   - Transaction isolation
   - Optimistic concurrency
   - Deadlock prevention

Crie o arquivo docs-ravendb-learning/05-Raven-Server.md com análise detalhada.
```

---

## Prompt 6: Raven.Client (Client Library)

```
Analise o projeto Raven.Client (src/Raven.Client) do RavenDB em profundidade, focando em:

1. **Session Management**
   - Unit of work pattern
   - Change tracking
   - Identity map
   - Lazy loading

2. **Caching Strategies**
   - Aggressive caching
   - Cache invalidation
   - Local caching
   - HTTP caching headers

3. **Network Protocol**
   - HTTP/2 usage
   - Request batching
   - Compression
   - Connection pooling

4. **Serialization**
   - JSON serialization
   - Type handling
   - Custom converters
   - Performance optimization

5. **Query API**
   - LINQ provider
   - Query generation
   - Result materialization
   - Projections

Crie o arquivo docs-ravendb-learning/06-Raven-Client.md com análise detalhada.
```

---

## Prompt 7: Raven.Embedded

```
Analise o projeto Raven.Embedded (src/Raven.Embedded) do RavenDB, focando em:

1. **Embedded Mode Architecture**
   - In-process server
   - Lifecycle management
   - Resource isolation

2. **Integration Patterns**
   - API surface
   - Configuration
   - Testing support

Crie o arquivo docs-ravendb-learning/07-Raven-Embedded.md com análise.
```

---

## Prompt 8: Raven.TestDriver

```
Analise o projeto Raven.TestDriver (src/Raven.TestDriver), focando em:

1. **Testing Infrastructure**
   - Test server management
   - Fixtures e helpers
   - Data seeding

2. **Best Practices**
   - Test isolation
   - Performance testing
   - Integration testing

Crie o arquivo docs-ravendb-learning/08-Raven-TestDriver.md com análise.
```

---

## Prompt 9-14: Análises Temáticas Cruzadas

```
Com base em todos os projetos analisados anteriormente, crie análises temáticas que cruzam todos os componentes:

9. **Performance-Patterns.md** - Todos os padrões de performance identificados
10. **Memory-Management.md** - Estratégias de gerenciamento de memória em todo o codebase
11. **Concurrency-Patterns.md** - Padrões de concorrência e sincronização
12. **Unsafe-Code.md** - Uso e padrões de código unsafe
13. **Data-Structures.md** - Estruturas de dados customizadas
14. **IO-Optimization.md** - Todas as otimizações de I/O

Para cada tema, cruze informações de todos os projetos, mostrando como diferentes componentes implementam a mesma técnica e quais variações existem.
```

---

## Prompt 15: Benchmark Analysis

```
Analise os projetos de benchmark (bench/*) do RavenDB:

1. **Benchmark Projects**
   - Micro.Benchmark
   - Indexing.Benchmark
   - BulkInsert.Benchmark
   - TimeSeries.Benchmark
   - Vector.Benchmark
   - Voron.Benchmark
   - Subscriptions.Benchmark
   - VxSort.Benchmark

2. **Para Cada Benchmark**
   - O que está sendo medido
   - Metodologia
   - Técnicas de medição precisa
   - Anti-patterns evitados

3. **BenchmarkDotNet Usage**
   - Configurações
   - Atributos usados
   - Análise de resultados

Crie o arquivo docs-ravendb-learning/15-Benchmark-Analysis.md
```

---

## Prompt 16: Testing Strategies

```
Analise as estratégias de teste no RavenDB:

1. **Test Projects**
   - FastTests
   - SlowTests
   - StressTests
   - RachisTests
   - BenchmarkTests

2. **Testing Patterns**
   - Test organization
   - Fixtures e setup
   - Parallel execution
   - Isolation strategies

3. **Performance Testing**
   - Load testing
   - Stress testing
   - Regression testing

Crie o arquivo docs-ravendb-learning/16-Testing-Strategies.md
```

---

## ?? Ordem Recomendada de Execução

### Nível 1: Fundações (Começar aqui)
1. Sparrow (base de performance)
2. Voron (storage)
3. Sparrow.Server (server utilities)

### Nível 2: Features Core
4. Corax (search)
5. Raven.Server (orchestration)
6. Raven.Client (client patterns)

### Nível 3: Suporte
7. Raven.Embedded
8. Raven.TestDriver

### Nível 4: Síntese
9-14. Análises temáticas
15. Benchmarks
16. Testing

---

## ?? Dicas

- **Execute um prompt por vez** para análise mais profunda
- **Faça perguntas de follow-up** sobre partes específicas
- **Compare implementações** entre projetos
- **Pratique os padrões** identificados em projetos próprios
- **Documente suas próprias descobertas** adicionando ao markdown

---

## ?? Estimativa de Tempo

- Cada análise de projeto: 10-20 minutos
- Análises temáticas: 15-30 minutos cada
- Total completo: 3-5 horas de análise

**Sugestão:** Faça 2-3 análises por dia para absorver melhor o conteúdo.
