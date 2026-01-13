# Voron - Storage Engine de Alta Performance

## ?? Visão Geral

**Voron** é o storage engine transacional de baixo nível do RavenDB. É um **embedded transactional storage engine** que implementa propriedades ACID completas com altíssima performance através de técnicas avançadas de gerenciamento de memória e I/O.

### Propósito
- **Storage Engine** ACID-compliant para dados persistentes
- **B+Tree** implementation otimizada para alta performance
- **Write-Ahead Logging (WAL)** para durabilidade
- **Memory-mapped files** para acesso rápido aos dados
- **Copy-on-Write (CoW)** semantics para transações MVCC

### Arquitetura em Camadas

```
???????????????????????????????????????????????????????
?           Transaction API (High-Level)               ?
?  (Tree, Table, FixedSizeTree, Container, Lookup)   ?
???????????????????????????????????????????????????????
?          Storage Structures (Mid-Level)              ?
?  • B+Trees (Variable & Fixed)                       ?
?  • Compact Trees • Tables • Lookups                 ?
?  • Posting Lists • Containers                       ?
???????????????????????????????????????????????????????
?        Transaction Management (Low-Level)            ?
?  • LowLevelTransaction • TransactionPersistentContext?
?  • MVCC • Snapshot Isolation                        ?
???????????????????????????????????????????????????????
?             Journal & Recovery (WAL)                 ?
?  • WriteAheadJournal • JournalApplicator            ?
?  • Journal Files • Transaction Headers              ?
???????????????????????????????????????????????????????
?           Memory & Page Management                   ?
?  • AbstractPager (Data, Journal, Scratch)           ?
?  • ScratchBufferPool • DecompressionBuffers         ?
?  • Memory-Mapped Files • Page Allocation            ?
???????????????????????????????????????????????????????
?              Platform Abstraction                    ?
?  • Windows (Memory-Mapped) • Posix (mmap)          ?
?  • 32-bit vs 64-bit handling                        ?
???????????????????????????????????????????????????????
```

---

## ?? Técnicas de Alta Performance

### 1. Memory-Mapped Files & Page Management

#### **WindowsMemoryMapPager** - Memory Mapping no Windows
```csharp
public sealed unsafe class WindowsMemoryMapPager : AbstractPager
{
    public const int AllocationGranularity = 64 * Constants.Size.Kilobyte;
    private long _totalAllocationSize;
    private readonly FileStream _fileStream;
    private readonly SafeFileHandle _handle;
    
    // Memory mapping com granularidade de 64KB
    // Aloca em múltiplos de 64KB para eficiência do OS
}
```

**Técnicas:**
- ? **Memory-Mapped Files** para acesso rápido ao disco
- ? **Granularidade de 64KB** alinhada com requisitos do OS
- ? **Copy-on-Write mode** para operações de leitura otimizadas
- ? **Encryption support** integrado no pager

#### **Page Allocation Strategy**
```csharp
// StorageEnvironment - Page allocation tracking
private readonly long[] _validPagesAfterLoad;
private readonly long _lastValidPageAfterLoad;

[MethodImpl(MethodImplOptions.AggressiveInlining)]
public unsafe void ValidatePageChecksum(long pageNumber, PageHeader* current)
{
    // Fast path: páginas além do tamanho inicial não precisam validação
    if (pageNumber >= _lastValidPageAfterLoad)
        return;

    var index = pageNumber / (8 * sizeof(long));
    long old = _validPagesAfterLoad[index];
    var bitToSet = 1L << (int)(pageNumber % (8 * sizeof(long)));
    
    if ((old & bitToSet) != 0)
        return; // Já validada
        
    UnlikelyValidatePage(pageNumber, current, index, old, bitToSet);
}
```

**Técnicas:**
- ? **Bitmap tracking** para páginas validadas (1 bit por página)
- ? **Lazy validation** - valida apenas na primeira leitura
- ? **Checksum validation** com XXHash64
- ? **Lock-free bitmap** com CompareExchange

---

### 2. Transaction Management (MVCC)

#### **StorageEnvironment** - Transaction Orchestration
```csharp
public sealed class StorageEnvironment : IDisposable
{
    private readonly SemaphoreSlim _transactionWriter = new SemaphoreSlim(1, 1);
    private readonly AsyncManualResetEvent _writeTransactionRunning = new AsyncManualResetEvent();
    private readonly ThreadHoppingReaderWriterLock FlushInProgressLock = new ThreadHoppingReaderWriterLock();
    private readonly ReaderWriterLockSlim _txCreation = new ReaderWriterLockSlim();
    
    internal LowLevelTransaction NewLowLevelTransaction(
        TransactionPersistentContext transactionPersistentContext, 
        TransactionFlags flags, 
        ByteStringContext context = null, 
        TimeSpan? timeout = null)
    {
        bool txLockTaken = false;
        bool flushInProgressReadLockTaken = false;
        
        try
        {
            if (flags == TransactionFlags.ReadWrite)
            {
                // Previne flush durante criação de write transaction
                flushInProgressReadLockTaken = FlushInProgressLock.TryEnterReadLock(wait);
                
                // Apenas 1 write transaction por vez
                txLockTaken = _transactionWriter.Wait(wait);
                
                _currentWriteTransactionHolder = NativeMemory.CurrentThreadStats;
                WriteTransactionStarted();
            }
            
            // Previne novas transações durante operações críticas
            _txCreation.EnterReadLock();
            try
            {
                long txId = flags == TransactionFlags.ReadWrite 
                    ? NextWriteTransactionId 
                    : CurrentReadTransactionId;
                    
                var tx = new LowLevelTransaction(this, txId, ...);
                ActiveTransactions.Add(tx);
                return tx;
            }
            finally
            {
                _txCreation.ExitReadLock();
            }
        }
        catch (Exception)
        {
            // Cleanup em caso de erro
            if (txLockTaken)
                _transactionWriter.Release();
            if (flushInProgressReadLockTaken)
                FlushInProgressLock.ExitReadLock();
            throw;
        }
    }
}
```

**Técnicas:**
- ? **Single-writer, multiple-readers** (MVCC)
- ? **Hierarchical locking** (flush lock > write tx lock)
- ? **ThreadHoppingReaderWriterLock** para prevenir thread-hopping no async
- ? **Active transactions tracking** para determinar oldest active tx
- ? **Snapshot isolation** - readers nunca bloqueiam writers

#### **Transaction** - High-Level API
```csharp
public sealed unsafe class Transaction : IDisposable
{
    private LowLevelTransaction _lowLevelTransaction;
    private Dictionary<Slice, Tree> _trees;
    private Dictionary<TableKey, Table> _tables;
    private Dictionary<ContainerId, Container.TransactionState> _containers;
    
    // Cache de estruturas abertas para evitar realocações
    private Dictionary<long, ByteString> _cachedDecompressedBuffersByStorageId;
    
    internal void PrepareForCommit()
    {
        OnBeforeCommit?.Invoke(this);
        
        // Persiste todos os multi-value trees
        if (_multiValueTrees != null)
        {
            foreach (var multiValueTree in _multiValueTrees)
            {
                var parentTree = multiValueTree.Key.Item1;
                var childTree = multiValueTree.Value;
                
                using (parentTree.DirectAdd(key, sizeof(TreeRootHeader), 
                    TreeNodeFlags.MultiValuePageRef, out byte* ptr))
                {
                    childTree.State.CopyTo((TreeRootHeader*)ptr);
                }
            }
        }
        
        // Atualiza todas as árvores modificadas
        foreach (var tree in Trees)
        {
            if (tree?.State.IsModified == true)
            {
                using (_lowLevelTransaction.RootObjects.DirectAdd(
                    tree.Name, sizeof(TreeRootHeader), out byte* ptr))
                {
                    tree.State.CopyTo((TreeRootHeader*)ptr);
                }
            }
        }
        
        _lowLevelTransaction.PrepareForCommit();
    }
}
```

**Técnicas:**
- ? **Caching de estruturas** para evitar re-opening
- ? **Lazy materialization** de trees/tables
- ? **Two-phase commit** (prepare + commit)
- ? **Event-based hooks** (OnBeforeCommit)

---

### 3. B+Tree Implementation

#### **Tree** - Variable Size B+Tree
```csharp
public unsafe partial class Tree
{
    private readonly TreeMutableState _state;
    private readonly LowLevelTransaction _llt;
    
    // Cache de páginas recentemente encontradas
    private RecentlyFoundTreePages _recentlyFoundPages;
    
    // Pool para evitar alocações
    private static readonly ObjectPool<RecentlyFoundTreePages> FoundPagesPool = 
        new(() => new RecentlyFoundTreePages(), 128);
    
    public DirectAddScope DirectAdd(Slice key, int len, out byte* ptr)
    {
        var foundPage = FindPageFor(key, out TreeNodeHeader* node, 
            out TreeCursorConstructor cursorConstructor, allowCompressed: true);
        
        if (foundPage.LastMatch == 0) // Update operation
        {
            node = foundPage.GetNode(foundPage.LastSearchPosition);
            
            // Otimização: tenta sobrescrever no mesmo espaço
            if (TryOverwriteDataOrMultiValuePageRefNode(node, len, nodeType, out pos))
            {
                ptr = pos;
                return new DirectAddScope(this);
            }
            
            RemoveLeafNode(foundPage);
        }
        else // Insert operation
        {
            State.Modify().NumberOfEntries++;
        }
        
        // Verifica se precisa de overflow page
        if (ShouldGoToOverflowPage(len))
        {
            pageNumber = WriteToOverflowPages(len, out overFlowPos);
            len = -1;
            nodeType = TreeNodeFlags.PageRef;
        }
        
        // Page split se necessário
        if (page.HasSpaceFor(_llt, key, len) == false)
        {
            if (IsLeafCompressionSupported == false || 
                TryCompressPageNodes(key, len, page) == false)
            {
                var pageSplitter = new TreePageSplitter(_llt, this, key, len, 
                    pageNumber, nodeType, cursor);
                dataPos = pageSplitter.Execute();
                
                ptr = overFlowPos ?? dataPos;
                return new DirectAddScope(this);
            }
        }
        
        // Adiciona na página
        dataPos = page.AddDataNode(lastSearchPosition, key, len);
        ptr = overFlowPos ?? dataPos;
        return new DirectAddScope(this);
    }
}
```

**Técnicas:**
- ? **Direct pointer access** com `DirectAdd` para zero-copy writes
- ? **In-place updates** quando possível
- ? **Page compression** para leaf pages
- ? **Overflow pages** para valores grandes
- ? **Object pooling** para caches de páginas
- ? **Copy-on-Write** semantics

#### **RecentlyFoundTreePages** - Page Cache
```csharp
private class RecentlyFoundTreePages
{
    // Cache das últimas páginas acessadas para evitar re-search
    private struct CachedPage
    {
        public TreePage Page;
        public long Number;
        public SliceOptions FirstKeyOption;
        public ReadOnlySpan<byte> FirstKey;
        public SliceOptions LastKeyOption;
        public ReadOnlySpan<byte> LastKey;
        public long[] Cursor; // Path to page
    }
    
    public bool TryFind(Slice key, out CachedPage foundPage)
    {
        // Busca linear no cache (pequeno, ~16 entries)
        // Verifica se a key está no range [FirstKey, LastKey]
        // Retorna a página e o cursor completo
    }
}
```

**Técnicas:**
- ? **Page cache** para evitar tree walks repetidos
- ? **Range-based lookup** (FirstKey, LastKey)
- ? **Cursor caching** para rápida re-navegação
- ? **Small fixed-size cache** para melhor locality

---

### 4. Write-Ahead Journal (WAL)

#### **WriteAheadJournal** - Durability & Recovery
```csharp
public sealed unsafe class WriteAheadJournal
{
    private ImmutableAppendOnlyList<JournalFile> _files;
    private JournalFile CurrentFile;
    private readonly JournalApplicator _journalApplicator;
    
    // Compression buffer para transações grandes
    private AbstractPager _compressionPager;
    private readonly DiffPages _diffPage = new DiffPages();
    
    public CompressedPagesResult WriteToJournal(LowLevelTransaction tx)
    {
        lock (_writeLock)
        {
            var journalEntry = PrepareToWriteToJournal(tx, 
                out var numberOfUsedCompressionBufferPages);
            
            // Cria novo journal se necessário
            if (CurrentFile == null || 
                CurrentFile.Available4Kbs < journalEntry.NumberOf4Kbs)
            {
                CurrentFile = NextFile(journalEntry.NumberOf4Kbs);
            }
            
            // Escreve para o journal
            CurrentFile.Write(tx, journalEntry);
            
            // Zera buffer de compressão (segurança)
            if (_env.Options.Encryption.IsEnabled)
                ZeroCompressionBuffer(tx);
            
            return journalEntry;
        }
    }
    
    private CompressedPagesResult PrepareToWriteToJournal(
        LowLevelTransaction tx, 
        out int totalNumberOfUsedCompressionBufferPages)
    {
        var txPages = tx.GetTransactionPages();
        var performCompression = pagesCount > 
            _env.Options.CompressTxAboveSizeInBytes / Constants.Storage.PageSize;
        
        // Calcula overhead de headers
        var sizeOfPagesHeader = numberOfPages * sizeof(TransactionHeaderPageInfo);
        var overhead = sizeOfPagesHeader + (long)numberOfPages * sizeof(long);
        
        // Aplica diff compression se habilitado
        if (performCompression && !_env.Options.Encryption.IsEnabled)
        {
            foreach (var txPage in txPages)
            {
                if (txPage.PreviousVersion != null)
                {
                    // Diff compression
                    _diffPage.ComputeDiff(
                        txPage.PreviousVersion.Value.Pointer, 
                        scratchPage, 
                        diffPageSize);
                }
                else
                {
                    // New page compression
                    _diffPage.ComputeNew(scratchPage, diffPageSize);
                }
                
                transactionHeaderPageInfo.DiffSize = _diffPage.OutputSize;
            }
            
            // LZ4 compression do resultado
            compressedLen = LZ4.Encode64LongBuffer(
                txPageInfoPtr,
                compressionBuffer,
                totalSizeWritten,
                outputBufferSize,
                compressionAcceleration);
        }
        
        // Calcula hash da transação
        txHeader.Hash = Hashing.XXHash64.Calculate(
            compressionBuffer, 
            (ulong)compressedLen, 
            (ulong)txHeader.TransactionId);
        
        return prepareToWriteToJournal;
    }
}
```

**Técnicas:**
- ? **Write-Ahead Logging** para durabilidade ACID
- ? **Diff compression** entre versões de páginas
- ? **LZ4 compression** para reduzir I/O
- ? **Adaptive compression acceleration** baseado em performance
- ? **Encryption support** com AEAD (XChaCha20-Poly1305)
- ? **Checksum validation** (XXHash64)

#### **JournalApplicator** - Applying Journals to Data File
```csharp
public sealed class JournalApplicator
{
    private LastFlushState _lastFlushed;
    private readonly object _flushingLock = new object();
    
    public void ApplyLogsToDataFile(CancellationToken token, TimeSpan timeToWait)
    {
        Monitor.TryEnter(_flushingLock, timeToWait, ref lockTaken);
        
        try
        {
            var jrnls = GetJournalSnapshots();
            if (jrnls.Count == 0)
                return;
            
            var pagesToWrite = new Dictionary<long, PagePosition>();
            long oldestActiveTransaction = 
                _waj._env.ActiveTransactions.OldestTransaction;
            
            // Coleta páginas modificadas de todos os journals
            foreach (var journalFile in jrnls)
            {
                var maxTransactionId = journalFile.LastTransaction;
                if (oldestActiveTransaction != 0)
                    maxTransactionId = Math.Min(
                        oldestActiveTransaction - 1, 
                        maxTransactionId);
                
                foreach (var modifiedPagesInTx in 
                    journalFile.PageTranslationTable
                        .GetModifiedPagesForTransactionRange(
                            lastFlushed.TransactionId, 
                            maxTransactionId))
                {
                    foreach (var pagePosition in modifiedPagesInTx)
                    {
                        if (pagePosition.Value.IsFreedPageMarker)
                        {
                            pagesToWrite.Remove(pagePosition.Key);
                            continue;
                        }
                        
                        pagesToWrite[pagePosition.Key] = pagePosition.Value;
                    }
                }
            }
            
            // Aplica páginas do scratch buffer para o data file
            ApplyPagesToDataFileFromScratch(pagesToWrite);
            
            // Atualiza estado do journal de forma atômica
            ApplyJournalStateAfterFlush(token, jrnls, 
                lastProcessedJournal, lastFlushedTransactionId);
        }
        finally
        {
            if (lockTaken)
                Monitor.Exit(_flushingLock);
        }
    }
    
    private void ApplyPagesToDataFileFromScratch(
        Dictionary<long, PagePosition> pagesToWrite)
    {
        using (var batchWrites = _waj._dataPager.BatchWriter())
        {
            foreach (var pagePosition in pagesToWrite.Values)
            {
                // Valida checksum antes de escrever
                var page = scratchBufferPool.AcquirePagePointerWithOverflowHandling(
                    tempTx, scratchNumber, pagePosition.ScratchPage, pagerState);
                    
                var checksum = StorageEnvironment.CalculatePageChecksum(
                    (byte*)page, page->PageNumber, 
                    out var expectedChecksum);
                    
                if (checksum != expectedChecksum)
                    ThrowInvalidChecksumOnPageFromScratch(...);
                
                // Copy para data file
                var numberOfPages = scratchBufferPool.CopyPage(
                    batchWrites,
                    scratchNumber,
                    pagePosition.ScratchPage,
                    pagerState);
            }
        }
    }
}
```

**Técnicas:**
- ? **Asynchronous journal application** não bloqueia writes
- ? **Oldest active transaction tracking** para determinar safe point
- ? **Batch writes** para otimizar I/O
- ? **Checksum validation** antes de aplicar páginas
- ? **Two-phase state update** (flush + update header)

---

### 5. Page Compression

#### **LeafPageCompressor** - Compressão de Leaf Pages
```csharp
public static class LeafPageCompressor
{
    public static DecompressedLeafPage Decompress(
        LowLevelTransaction llt,
        TreePage page,
        DecompressionUsage usage,
        bool skipCache)
    {
        // Descomprime LZ4 para buffer temporário
        var decompressedPage = new DecompressedLeafPage(
            page, 
            decompressedBuffer, 
            numberOfEntries);
        
        // Cache se for para leitura
        if (!skipCache && usage == DecompressionUsage.Read)
        {
            llt.Environment.DecompressionBuffers.AddDecompressedPage(
                decompressedPage);
        }
        
        return decompressedPage;
    }
}
```

**Técnicas:**
- ? **LZ4 compression** para leaf pages
- ? **Decompression cache** para reads repetidos
- ? **Lazy decompression** - apenas quando acessado
- ? **Skip cache** option para scans únicos

---

### 6. Scratch Buffer Pool

#### **ScratchBufferPool** - Temporary Page Storage
```csharp
public sealed class ScratchBufferPool
{
    private readonly ConcurrentDictionary<int, ScratchBufferFile> _scratchBuffers;
    
    public Page ReadPage(
        LowLevelTransaction tx, 
        int scratchNumber, 
        long positionInScratchBuffer, 
        PagerState pagerState = null)
    {
        var scratchBuffer = GetScratchBufferFile(scratchNumber);
        
        if (pagerState == null)
            pagerState = scratchBuffer.File.Pager.GetPagerStateAndAddRefAtomically();
        
        var pagePointer = scratchBuffer.File.Pager.AcquirePagePointer(
            tx, 
            positionInScratchBuffer);
        
        return new Page(pagePointer);
    }
}
```

**Técnicas:**
- ? **Scratch buffers** para páginas temporárias (não-durável)
- ? **Numbered scratch files** para isolamento
- ? **Pager state management** com reference counting
- ? **Cleanup basead em oldest active transaction**

---

### 7. Encryption (AEAD)

#### **Encryption Support**
```csharp
private void EncryptTransaction(byte* fullTxBuffer)
{
    var txHeader = (TransactionHeader*)fullTxBuffer;
    txHeader->Flags |= TransactionPersistenceModeFlags.Encrypted;
    
    // Deriva sub-key do transaction ID
    var subKey = stackalloc byte[(int)subKeyLen];
    fixed (byte* mk = _env.Options.Encryption.MasterKey)
    {
        Sodium.crypto_kdf_derive_from_key(
            subKey, 
            subKeyLen, 
            (ulong)txHeader->TransactionId, 
            ctx, 
            mk);
    }
    
    // Gera nonce aleatório
    var npub = fullTxBuffer + TransactionHeader.NonceOffset;
    Sodium.randombytes_buf(npub, (UIntPtr)TransactionHeader.NonceSize);
    
    // Encripta com XChaCha20-Poly1305 (AEAD)
    var rc = Sodium.crypto_aead_xchacha20poly1305_ietf_encrypt_detached(
        fullTxBuffer + TransactionHeader.SizeOf,  // ciphertext
        fullTxBuffer + TransactionHeader.SizeOf - macLen,  // MAC
        &macLen,
        fullTxBuffer + TransactionHeader.SizeOf,  // plaintext
        (ulong)size,
        fullTxBuffer,  // associated data (header)
        (ulong)(TransactionHeader.SizeOf - TransactionHeader.NonceOffset),
        null,
        npub,  // nonce
        subKey);  // key
}
```

**Técnicas:**
- ? **AEAD encryption** (XChaCha20-Poly1305)
- ? **Key derivation** por transaction (KDF)
- ? **Random nonces** para cada transação
- ? **Authenticated encryption** elimina necessidade de checksums separados
- ? **Memory zeroing** de buffers sensíveis

---

### 8. Recovery

#### **Journal Recovery**
```csharp
public bool RecoverDatabase(
    TransactionHeader* txHeader, 
    Action<LogLevel, string> addToInitLog)
{
    var logInfo = _headerAccessor.Get(ptr => ptr->Journal);
    var modifiedPages = new HashSet<long>();
    
    var journalToStartReadingFrom = logInfo.LastSyncedJournal;
    
    // Itera sobre todos os journals não aplicados
    for (var journalNumber = journalToStartReadingFrom; 
         journalNumber <= logInfo.CurrentJournal; 
         journalNumber++)
    {
        using (var journalReader = new JournalReader(pager, _dataPager, 
            recoveryPager, modifiedPages, logInfo, currentFileHeader, null))
        {
            var transactionHeaders = journalReader.RecoverAndValidate(
                _env.Options);
            
            var lastReadHeaderPtr = journalReader.LastTransactionHeader;
            if (lastReadHeaderPtr != null)
            {
                *txHeader = *lastReadHeaderPtr;
                lastFlushedTxId = txHeader->TransactionId;
                lastFlushedJournal = journalNumber;
            }
            
            var jrnlFile = new JournalFile(_env, jrnlWriter, journalNumber);
            jrnlFile.InitFrom(journalReader, transactionHeaders);
            
            journalFiles.Add(jrnlFile);
        }
    }
    
    // Valida checksums de todas as páginas modificadas
    if (_env.Options.SkipChecksumValidationOnDatabaseLoading == false)
    {
        for (var i = sortedPages.Length - 1; i >= 0; i--)
        {
            var ptr = (PageHeader*)_dataPager
                .AcquirePagePointerWithOverflowHandling(
                    tempTx, modifiedPage, null);
            
            _env.ValidateInMemoryPageChecksum(modifiedPage, ptr);
        }
    }
    
    return requireHeaderUpdate;
}
```

**Técnicas:**
- ? **Forward recovery** a partir do último sync
- ? **Checksum validation** de todas as páginas modificadas
- ? **Partial transaction handling** (não comitadas)
- ? **Header update** após recovery

---

## ?? Estruturas de Dados Especializadas

### 1. **B+Trees** (Variable & Fixed Size)
- **Tree** - Variable-size keys e values
- **FixedSizeTree** - Fixed-size values (otimizado)
- **CompactTree** - Compressed keys para menor overhead

### 2. **Tables** (Schema-based Storage)
- Schema-driven storage com índices
- Primary key support
- Secondary indexes (fixed & variable)
- Table sections para dados raw

### 3. **Lookups** (Key-Value Maps)
- Int64LookupKey
- DoubleLookupKey  
- CompactKeyLookup (string keys comprimidas)

### 4. **Containers** (Self-managed Pages)
- Free space tracking
- Variable-size allocation
- Used para Hnsw vectors

### 5. **PostingLists** (Inverted Indexes)
- Compressed posting lists
- Efficient intersection/union
- Used para full-text search

---

## ?? Código Notável

### 1. **Inline Checksum Calculation**
```csharp
public static unsafe ulong CalculatePageChecksum(
    byte* ptr, 
    long pageNumber, 
    PageFlags flags, 
    int overflowSize)
{
    var dataLength = Constants.Storage.PageSize - 
        (PageHeader.ChecksumOffset + sizeof(ulong));
        
    if ((flags & PageFlags.Overflow) == PageFlags.Overflow)
        dataLength = overflowSize - 
            (PageHeader.ChecksumOffset + sizeof(ulong));
    
    var ctx = new Hashing.Streamed.XXHash64Context
    {
        Seed = (ulong)pageNumber  // Page number como seed
    };
    
    Hashing.Streamed.XXHash64.Begin(ref ctx);
    
    // Hash antes do checksum field
    Hashing.Streamed.XXHash64.Process(ref ctx, ptr, 
        PageHeader.ChecksumOffset);
        
    // Hash depois do checksum field (pula o checksum)
    Hashing.Streamed.XXHash64.Process(ref ctx, 
        ptr + PageHeader.ChecksumOffset + sizeof(ulong), 
        dataLength);
    
    return Hashing.Streamed.XXHash64.End(ref ctx);
}
```

**Por que é notável:**
- ? Uses page number as seed para detectar page swaps
- ? Streaming hash para suportar páginas grandes
- ? Pula o campo checksum durante cálculo
- ? Suporta overflow pages

### 2. **Lock-Free Page Validation Bitmap**
```csharp
[MethodImpl(MethodImplOptions.AggressiveInlining)]
public unsafe void ValidatePageChecksum(long pageNumber, PageHeader* current)
{
    if (pageNumber >= _lastValidPageAfterLoad)
        return;  // New pages don't need validation
    
    var index = pageNumber / (8 * sizeof(long));
    long old = _validPagesAfterLoad[index];
    var bitToSet = 1L << (int)(pageNumber % (8 * sizeof(long)));
    
    if ((old & bitToSet) != 0)
        return;  // Already validated
    
    UnlikelyValidatePage(pageNumber, current, index, old, bitToSet);
}

private unsafe void UnlikelyValidatePage(
    long pageNumber, 
    PageHeader* current, 
    long index, 
    long old, 
    long bitToSet)
{
    ulong checksum = CalculatePageChecksum(...);
    if (checksum != current->Checksum)
        ThrowInvalidChecksum(...);
    
    // Lock-free bitmap update com CompareExchange
    while (true)
    {
        long modified = Interlocked.CompareExchange(
            ref _validPagesAfterLoad[index], 
            old | bitToSet, 
            old);
            
        if (modified == old || (modified & bitToSet) != 0)
            break;
        
        old = modified;
    }
}
```

**Por que é notável:**
- ? **Lock-free** com CompareExchange loop
- ? **Fast path inline** para páginas já validadas
- ? **Unlikely path separation** para melhor branch prediction
- ? **1 bit per page** para minimal memory overhead

### 3. **Adaptive Compression Acceleration**
```csharp
public void CalculateOptimalAcceleration()
{
    // Se compressão é MUITO maior que write time, aumenta aceleração
    if (CompressionDuration > WriteDuration.Add(WriteDuration))
    {
        if (_lastAcceleration < 99)
        {
            _lastAcceleration = Math.Min(99, _lastAcceleration + 2);
            _flux = -4;
        }
        return;
    }
    
    // Write time maior que compressão - vale a pena comprimir
    if (CompressionDuration <= WriteDuration)
    {
        if (++_flux > 5)  // Requer várias operações consecutivas
        {
            _lastAcceleration = Math.Max(1, _lastAcceleration - 1);
            _flux = 3;
        }
        return;
    }
    
    // Compressão maior que write - I/O rápido, menos compressão
    if (--_flux < -5)
    {
        _lastAcceleration = Math.Min(99, _lastAcceleration + 1);
        _flux = -2;
    }
}
```

**Por que é notável:**
- ? **Self-tuning** baseado em performance real
- ? **Flux control** para evitar oscilações
- ? **Adapta ao hardware** (fast SSD vs slow HDD)
- ? **LZ4 acceleration** (1 = max compression, 99 = max speed)

### 4. **Two-Phase Journal State Update**
```csharp
private void ApplyJournalStateAfterFlush(...)
{
    _waj._env.FlushInProgressLock.EnterWriteLock();
    
    try
    {
        var singleUseFlag = new SingleUseFlag();
        
        // Define ação para executar sob write tx lock
        Action<LowLevelTransaction> currentAction = txw =>
        {
            if (singleUseFlag.Raise() == false)
                throw new InvalidOperationException("Tried to update twice");
            
            try
            {
                UpdateJournalStateUnderWriteTransactionLock(
                    txw, journalSnapshots, lastProcessedJournal, 
                    lastFlushedTransactionId);
            }
            finally
            {
                _updateJournalStateAfterFlush = null;
                _waitForJournalStateUpdateUnderTx.Set();
            }
        };
        
        Interlocked.Exchange(ref _updateJournalStateAfterFlush, currentAction);
        
        // Tenta pegar write tx lock
        do
        {
            try
            {
                txw = _waj._env.NewLowLevelTransaction(..., timeout: TimeSpan.Zero);
            }
            catch (TimeoutException)
            {
                // Espera transação em andamento completar
                var satisfiedIndex = WaitHandle.WaitAny(new[] { 
                    _waitForJournalStateUpdateUnderTx.WaitHandle, 
                    _onWriteTransactionCompleted.WaitHandle, 
                    token.WaitHandle 
                }, TimeSpan.FromMilliseconds(250));
                
                continue;
            }
            
            var action = _updateJournalStateAfterFlush;
            if (action != null)
            {
                action(txw);  // Executa sob o lock
                txw.Commit();
            }
            break;
            
        } while (currentAction == _updateJournalStateAfterFlush);
    }
    finally
    {
        _waj._env.FlushInProgressLock.ExitWriteLock();
    }
}
```

**Por que é notável:**
- ? **Piggybacks** na próxima write transaction para evitar criar uma nova
- ? **Não bloqueia** writes desnecessariamente
- ? **Single-use flag** previne execução duplicada
- ? **Timeout handling** para evitar deadlocks

---

## ?? Lições Aprendidas

### 1. **MVCC Implementation**
- **Single-writer, multiple-readers** com snapshot isolation
- Write lock hierárquico (flush > write tx) para prevenir deadlocks
- Active transactions tracking para determinar safe GC point
- Copy-on-Write para zero-copy reads

### 2. **Memory-Mapped Files**
- Granularidade de 64KB alinhada com OS requirements
- Pager state management com reference counting
- Validation bitmap para lazy checksum checking
- Platform abstraction (Windows vs POSIX)

### 3. **Write-Ahead Logging**
- Diff compression entre versões de páginas
- Adaptive LZ4 compression acceleration
- Encryption com AEAD (XChaCha20-Poly1305)
- Asynchronous journal application

### 4. **B+Tree Optimization**
- Recently found pages cache para evitar tree walks
- Direct pointer API para zero-copy writes
- In-place updates quando possível
- Page compression para leaf pages

### 5. **Recovery & Durability**
- Forward recovery a partir do último sync
- Checksum validation de todas as páginas
- Partial transaction handling
- Journal cleanup baseado em oldest active tx

---

## ?? Performance Metrics

### **Allocation Strategies**
- **32-bit mode**: Power-of-2 allocation até 1MB, depois MB-aligned
- **64-bit mode**: Direct allocation
- **Scratch buffer pooling** para reuso
- **Compression buffer** com auto-reduce

### **Compression**
- **LZ4 acceleration**: 1 (max compression) to 99 (max speed)
- **Diff compression** para páginas modificadas
- **Adaptive tuning** baseado em CompressionTime vs WriteTime
- **Skip compression** para encrypted data (AEAD)

### **Caching**
- **Recently found pages** (16 entries típico)
- **Decompressed pages cache**
- **Scratch pager state cache**
- **Tree/Table/Container cache** em Transaction

---

## ?? Componentes Críticos

1. **StorageEnvironment** - Orchestration & lifecycle
2. **LowLevelTransaction** - MVCC implementation
3. **Tree** - B+Tree core
4. **WriteAheadJournal** - Durability
5. **JournalApplicator** - Async flush to data file
6. **AbstractPager** - Platform abstraction
7. **ScratchBufferPool** - Temporary storage

---

## ? Checklist de Técnicas

### Memory Management
- [x] Memory-mapped files
- [x] Object pooling (RecentlyFoundTreePages)
- [x] Pager state reference counting
- [x] Scratch buffer pooling
- [x] Compression buffer auto-reduce

### Concurrency
- [x] MVCC (Single-writer, multiple-readers)
- [x] Snapshot isolation
- [x] Hierarchical locking
- [x] Lock-free bitmap (page validation)
- [x] ThreadHoppingReaderWriterLock

### I/O Optimization
- [x] Batch writes
- [x] Write-ahead logging
- [x] Diff compression
- [x] LZ4 adaptive compression
- [x] Async journal application

### Data Structures
- [x] B+Trees (variable & fixed)
- [x] Compact trees
- [x] Tables
- [x] Lookups
- [x] Containers
- [x] Posting lists

### Security
- [x] AEAD encryption (XChaCha20-Poly1305)
- [x] Key derivation per transaction
- [x] Memory zeroing
- [x] Checksum validation

### Reliability
- [x] Checksum validation (XXHash64)
- [x] Journal recovery
- [x] Partial transaction handling
- [x] Overflow detection
- [x] Data integrity checks

---

## ?? Conclusão

Voron é um **storage engine de classe mundial** que demonstra:

1. **ACID Transactions** com MVCC e snapshot isolation
2. **Memory-Mapped Files** para acesso de alta performance
3. **Write-Ahead Logging** com compression e encryption
4. **B+Trees otimizados** com caching e direct pointer access
5. **Adaptive tuning** para diferentes workloads
6. **Platform abstraction** (Windows/POSIX, 32/64-bit)
7. **Recovery robusto** com checksum validation

É uma base sólida para RavenDB, demonstrando como implementar um storage engine moderno com todas as garantias ACID sem sacrificar performance.

**Próximo:** [03-Sparrow-Server.md](03-Sparrow-Server.md) - Extensões server-side
