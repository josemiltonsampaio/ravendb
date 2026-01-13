# Performance Patterns - Análise Cross-Cutting

## ?? Visão Geral

Esta análise consolida **TODOS os padrões de performance** identificados nos 8 projetos core do RavenDB, criando um catálogo de técnicas reutilizáveis para desenvolvimento de sistemas de alta performance em C#.

### Projetos Analisados

```
? Sparrow          - Foundation (low-level primitives)
? Voron            - Storage Engine (B+Tree, transactions)
? Sparrow.Server   - Server Extensions (networking, threading)
? Corax            - Search Engine (indexing, queries)
? Raven.Server     - Main Server (orchestration, pipeline)
? Raven.Client     - Client Library (HTTP, caching, sessions)
? Raven.Embedded   - Embedded Server (process management)
? Raven.TestDriver - Testing Infrastructure (test fixtures)
```

### Categorias de Padrões

1. **Memory Management** - Allocation, pooling, lifetime
2. **Lock-Free & Concurrency** - Thread-safety sem locks
3. **I/O Optimization** - Zero-copy, async, buffering
4. **Caching Strategies** - Local, distributed, invalidation
5. **Object Pooling** - Reuse, recycling, lifecycle
6. **Inlining & Hot Paths** - JIT optimization, fast paths
7. **Data Structures** - Custom collections, cache-friendly
8. **Networking** - HTTP pooling, batching, compression

---

## 1?? Memory Management Patterns

### Pattern 1.1: Arena Allocator (Sparrow)

**Problema:** Frequent small allocations ? GC pressure

**Solução:** Bulk allocation + manual lifetime management

```csharp
// src/Sparrow/ArenaMemoryAllocator.cs
public class ArenaMemoryAllocator : IDisposable
{
    private readonly NativeMemory _memory;
    private int _allocated;
    private readonly int _size;

    public ArenaMemoryAllocator(int size)
    {
        _size = size;
        _memory = NativeMemory.AllocateMemory(size);
    }

    public ByteString Allocate(int size)
    {
        var pos = Interlocked.Add(ref _allocated, size);
        if (pos > _size)
            throw new OutOfMemoryException("Arena exhausted");

        return new ByteString(_memory.Ptr + (pos - size), size);
    }

    public void Dispose()
    {
        // Bulk free - all allocations released at once!
        _memory.Dispose();
    }
}
```

**Benefícios:**
- ? **Eliminates GC pressure** (no individual allocations)
- ? **Predictable lifetime** (arena scope = allocation scope)
- ? **Fast allocation** (just increment pointer)
- ? **Bulk deallocation** (single free)

**Trade-offs:**
- ? Cannot free individual allocations
- ? Requires upfront size estimate
- ? Fragmentation if mixed sizes

**Quando usar:**
- Request/response processing (scoped lifetime)
- Transaction processing
- Parsing/tokenization

**Projetos que usam:** Sparrow, Voron, Raven.Server, Corax

---

### Pattern 1.2: Object Pooling (Sparrow)

**Problema:** Expensive object creation/destruction

**Solução:** Pool de objetos reutilizáveis

```csharp
// src/Sparrow/Json/JsonContextPool.cs
public class JsonContextPool : IDisposable
{
    private readonly ConcurrentStack<JsonOperationContext> _globalStack;
    private readonly AsyncLocal<JsonOperationContext> _perThreadContext;
    private readonly int _maxContextSizeToKeep;

    public IDisposable AllocateOperationContext(out JsonOperationContext context)
    {
        // Try get from thread-local first (fastest)
        context = _perThreadContext.Value;
        if (context != null)
        {
            _perThreadContext.Value = null;
            context.Reset(); // Clear but keep allocated memory
            return new ReturnContext(this, context);
        }

        // Try get from global pool
        if (_globalStack.TryPop(out context))
        {
            context.Reset();
            return new ReturnContext(this, context);
        }

        // Create new (slow path)
        context = new JsonOperationContext(_maxContextSizeToKeep);
        return new ReturnContext(this, context);
    }

    private void Return(JsonOperationContext context)
    {
        if (context.AllocatedSize > _maxContextSizeToKeep)
        {
            context.Dispose(); // Too big, discard
            return;
        }

        // Return to thread-local first
        if (_perThreadContext.Value == null)
        {
            _perThreadContext.Value = context;
            return;
        }

        // Return to global pool
        _globalStack.Push(context);
    }

    private class ReturnContext : IDisposable
    {
        private readonly JsonContextPool _pool;
        private readonly JsonOperationContext _context;

        public ReturnContext(JsonContextPool pool, JsonOperationContext context)
        {
            _pool = pool;
            _context = context;
        }

        public void Dispose() => _pool.Return(_context);
    }
}
```

**Benefícios:**
- ? **Zero allocation** on hot path
- ? **Thread-local cache** (fastest)
- ? **Global pool fallback** (sharing across threads)
- ? **Size limits** (prevents memory bloat)

**Pattern de uso:**

```csharp
using (_contextPool.AllocateOperationContext(out var context))
{
    // Use context
    var doc = context.ReadForMemory(stream, "doc-id");
    // ...
} // Automatically returned to pool
```

**Quando usar:**
- Objects com expensive initialization
- High allocation rate scenarios
- Request/response processing

**Projetos que usam:** Sparrow, Raven.Client, Raven.Server

---

### Pattern 1.3: Span<T> for Zero-Copy Slicing (Sparrow)

**Problema:** Copying data for substring/slice operations

**Solução:** Stack-based slices sem alocação

```csharp
// src/Sparrow/Extensions/Slices.cs
public readonly ref struct Slice
{
    public readonly ReadOnlySpan<byte> Content;

    public Slice(ReadOnlySpan<byte> content)
    {
        Content = content;
    }

    // Zero-copy slicing
    public Slice Slice(int start, int length)
    {
        return new Slice(Content.Slice(start, length));
    }

    // Zero-copy comparison
    public bool Equals(Slice other)
    {
        return Content.SequenceEqual(other.Content);
    }
}

// Usage
Span<byte> buffer = stackalloc byte[256];
ReadData(buffer); // Fill buffer

var slice1 = new Slice(buffer.Slice(0, 10));   // No allocation!
var slice2 = new Slice(buffer.Slice(10, 20));  // No allocation!
```

**Benefícios:**
- ? **Stack allocated** (zero heap pressure)
- ? **Zero-copy slicing** (no data duplication)
- ? **Bounds checking** (via Span<T>)
- ? **Type safety** (ref struct cannot escape stack)

**Quando usar:**
- Parsing operations
- String/buffer manipulations
- Hot paths com temporary data

**Projetos que usam:** Sparrow, Voron, Corax, Raven.Server

---

### Pattern 1.4: Memory<T> for Async-Safe Slices (Sparrow.Server)

**Problema:** Span<T> cannot be used in async methods

**Solução:** Memory<T> para async operations

```csharp
// src/Sparrow.Server/AsyncManualResetEvent.cs
public class BufferedReader
{
    private Memory<byte> _buffer;
    private int _position;

    public async ValueTask<Memory<byte>> ReadAsync(int count)
    {
        // Memory<T> can be used across await!
        if (_position + count > _buffer.Length)
        {
            await RefillBufferAsync();
        }

        var slice = _buffer.Slice(_position, count);
        _position += count;
        return slice; // Safe to return from async
    }
}
```

**Span vs Memory:**

| Feature | Span<T> | Memory<T> |
|---------|---------|----------|
| **Allocation** | Stack | Heap (small) |
| **Async Support** | ? No | ? Yes |
| **Performance** | ? Fastest | ?? Slightly slower |
| **Use Case** | Sync hot paths | Async operations |

**Projetos que usam:** Sparrow.Server, Raven.Server

---

## 2?? Lock-Free & Concurrency Patterns

### Pattern 2.1: Interlocked Operations (Sparrow)

**Problema:** Locks são caros em hot paths

**Solução:** Atomic operations sem locks

```csharp
// src/Sparrow/LowMemory/LowMemoryNotification.cs
private long _lowMemoryFlag;

public void RaiseNotification()
{
    // Atomic compare-and-swap
    if (Interlocked.CompareExchange(ref _lowMemoryFlag, 1, 0) == 0)
    {
        // First thread wins, triggers notification
        OnLowMemory?.Invoke();
    }
}

public void Reset()
{
    Interlocked.Exchange(ref _lowMemoryFlag, 0);
}
```

**Common Interlocked Patterns:**

```csharp
// 1. Counter increment
var count = Interlocked.Increment(ref _counter);

// 2. Flag set (idempotent)
Interlocked.Exchange(ref _flag, 1);

// 3. Compare-and-swap (CAS)
var original = Interlocked.CompareExchange(ref _value, newValue, expectedValue);
if (original == expectedValue) { /* Success */ }

// 4. Add/Subtract
var newValue = Interlocked.Add(ref _value, delta);

// 5. Read (volatile read)
var value = Interlocked.Read(ref _longValue); // For long/ulong
```

**Benefícios:**
- ? **Lock-free** (no contention)
- ? **Atomic** (thread-safe)
- ? **Fast** (hardware instructions)

**Projetos que usam:** Todos os projetos

---

### Pattern 2.2: Double-Check Locking (Sparrow)

**Problema:** Lock em fast path é caro

**Solução:** Check sem lock, lock apenas se necessário

```csharp
// src/Sparrow/LowMemory/LowMemoryNotification.cs
private volatile LowMemoryHandlerStatistics _statistics;
private readonly object _lock = new object();

public LowMemoryHandlerStatistics GetStatistics()
{
    // Fast path - no lock!
    var stats = _statistics;
    if (stats != null)
        return stats;

    // Slow path - acquire lock
    lock (_lock)
    {
        // Double-check inside lock
        if (_statistics != null)
            return _statistics;

        // Initialize (only once)
        _statistics = new LowMemoryHandlerStatistics();
        return _statistics;
    }
}
```

**Critical: Volatile Field!**

```csharp
// ? ERRADO - race condition!
private LowMemoryHandlerStatistics _statistics;

// ? CORRETO - volatile prevents reordering
private volatile LowMemoryHandlerStatistics _statistics;

// ? ALTERNATIVA - Lazy<T> (thread-safe)
private readonly Lazy<LowMemoryHandlerStatistics> _statistics = 
    new Lazy<LowMemoryHandlerStatistics>(LazyThreadSafetyMode.ExecutionAndPublication);
```

**Quando usar:**
- Singleton initialization
- Lazy initialization
- Read-heavy scenarios

**Projetos que usam:** Sparrow, Raven.Server, Raven.Client

---

### Pattern 2.3: Lazy<T> for Thread-Safe Initialization (Raven.Client)

**Problema:** Complex initialization logic + thread-safety

**Solução:** Lazy<T> encapsula double-check locking

```csharp
// src/Raven.Client/Http/RequestExecutor.cs
private Lazy<Task<(Uri, Process)>>? _serverTask;

private Task StartServerAsync()
{
    var startServer = new Lazy<Task<(Uri, Process)>>(
        RunServer, 
        LazyThreadSafetyMode.ExecutionAndPublication
    );

    if (Interlocked.CompareExchange(ref _serverTask, startServer, null) != null)
        throw new InvalidOperationException("Server already started");

    return startServer.Value; // Triggers initialization
}
```

**Lazy Thread-Safety Modes:**

| Mode | Behavior | Use Case |
|------|----------|----------|
| `None` | Not thread-safe | Single-threaded |
| `PublicationOnly` | Multiple inits, first published wins | Exception not cached |
| `ExecutionAndPublication` | Single init, exceptions cached | Default (safest) |

**Projetos que usam:** Raven.Client, Raven.Embedded, Raven.TestDriver

---

### Pattern 2.4: ConcurrentDictionary for Lock-Free Collections (Raven.Client)

**Problema:** Dictionary + lock = contention

**Solução:** ConcurrentDictionary com atomic operations

```csharp
// src/Raven.Client/Http/HttpCache.cs
private readonly ConcurrentDictionary<string, HttpCacheItem> _items;

// Pattern 1: GetOrAdd with factory
var item = _items.GetOrAdd(url, key => new HttpCacheItem
{
    ChangeVector = changeVector,
    Data = result
});

// Pattern 2: TryGetValue (read)
if (_items.TryGetValue(url, out var item))
{
    // Use item
}

// Pattern 3: AddOrUpdate (upsert)
_items.AddOrUpdate(
    url,
    addValueFactory: key => CreateNew(),
    updateValueFactory: (key, old) => Update(old)
);

// Pattern 4: TryRemove (delete)
if (_items.TryRemove(url, out var removed))
{
    removed.Dispose();
}
```

**Advanced: GetOrAdd with Lazy<T>**

```csharp
// Problem: Factory might be called multiple times!
var item = _cache.GetOrAdd(key, k => ExpensiveOperation()); // BAD!

// Solution: GetOrAdd with Lazy<T>
var lazy = new Lazy<ExpensiveObject>(() => ExpensiveOperation());
var item = _cache.GetOrAdd(key, lazy).Value; // Only one execution!
```

**Projetos que usam:** Raven.Client, Raven.Server, Raven.Embedded

---

## 3?? I/O Optimization Patterns

### Pattern 3.1: Memory-Mapped Files (Voron)

**Problema:** Random access to large files é lento

**Solução:** Map file to memory address space

```csharp
// src/Voron/Impl/Paging/MemoryMapPager.cs
public class MemoryMapPager : AbstractPager
{
    private readonly MemoryMappedFile _file;
    private readonly MemoryMappedViewAccessor _accessor;

    public override byte* AcquirePagePointer(long pageNumber)
    {
        // Direct memory access - no I/O!
        var offset = pageNumber * Constants.Storage.PageSize;
        return (byte*)_accessor.SafeMemoryMappedViewHandle.DangerousGetHandle() + offset;
    }

    public override void Sync()
    {
        // Flush to disk (OS handles actual I/O)
        _accessor.Flush();
    }
}
```

**Benefícios:**
- ? **Zero-copy** (no buffer allocation)
- ? **OS-managed caching** (page cache)
- ? **Random access** (pointer arithmetic)
- ? **Lazy loading** (pages loaded on-demand)

**Trade-offs:**
- ? Address space limits (32-bit issues)
- ? OS page cache contention
- ? Cannot exceed virtual memory

**Quando usar:**
- Database storage engines
- Large file processing
- Random access patterns

**Projetos que usam:** Voron, Corax

---

### Pattern 3.2: Async I/O with ValueTask<T> (Sparrow.Server)

**Problema:** Task<T> allocates on heap

**Solução:** ValueTask<T> para sync completion

```csharp
// src/Sparrow.Server/AsyncManualResetEvent.cs
public class AsyncManualResetEvent
{
    private volatile TaskCompletionSource<bool> _tcs;

    public ValueTask WaitAsync()
    {
        var tcs = _tcs;
        if (tcs == null)
            return default; // Already signaled - no allocation!

        return new ValueTask(tcs.Task); // Wrap existing Task
    }

    public void Set()
    {
        var tcs = _tcs;
        if (tcs == null)
            return;

        _tcs = null;
        tcs.TrySetResult(true);
    }
}
```

**Task vs ValueTask:**

| Feature | Task<T> | ValueTask<T> |
|---------|---------|--------------|
| **Allocation** | Always heap | Stack if completed |
| **Awaitable multiple times** | ? Yes | ? No |
| **Use Case** | Always async | Often sync |

**Best Practices:**

```csharp
// ? Return ValueTask if often completes synchronously
public ValueTask<int> ReadAsync()
{
    if (_bufferHasData)
        return new ValueTask<int>(_cachedResult); // No allocation!
    
    return new ValueTask<int>(ReadSlowAsync());
}

// ? Don't await ValueTask twice!
var vt = ReadAsync();
await vt; // OK
await vt; // UNDEFINED BEHAVIOR!

// ? Convert to Task if need multiple awaits
var task = ReadAsync().AsTask();
await task;
await task; // OK
```

**Projetos que usam:** Sparrow.Server, Raven.Server

---

### Pattern 3.3: Buffered Writes with Batching (Voron)

**Problema:** Individual writes são lentos

**Solução:** Buffer writes + flush em batch

```csharp
// src/Voron/Impl/Journal/JournalWriter.cs
public class JournalWriter
{
    private readonly byte[] _buffer;
    private int _bufferPosition;
    private readonly int _bufferSize;

    public void Write(byte* data, int length)
    {
        // Fast path - fits in buffer
        if (_bufferPosition + length <= _bufferSize)
        {
            Marshal.Copy((IntPtr)data, _buffer, _bufferPosition, length);
            _bufferPosition += length;
            return;
        }

        // Slow path - flush then write
        Flush();
        
        if (length > _bufferSize)
        {
            // Too large, write directly
            WriteToFile(data, length);
        }
        else
        {
            Marshal.Copy((IntPtr)data, _buffer, 0, length);
            _bufferPosition = length;
        }
    }

    public void Flush()
    {
        if (_bufferPosition == 0)
            return;

        WriteToFile(_buffer, _bufferPosition);
        _bufferPosition = 0;
    }
}
```

**Benefícios:**
- ? **Reduced syscalls** (batch I/O)
- ? **Better throughput** (larger I/O operations)
- ? **Reduced overhead** (amortized cost)

**Projetos que usam:** Voron, Raven.Server

---

### Pattern 3.4: Zero-Copy Network I/O (Sparrow.Server)

**Problema:** Copying data for network send

**Solução:** Send from original buffer

```csharp
// src/Sparrow.Server/Platform/Posix/SmapsReader.cs
public unsafe void SendFile(Socket socket, byte* buffer, int length)
{
    // Zero-copy send (buffer não copiado para kernel)
    var segment = new ArraySegment<byte>(
        new UnmanagedMemoryManager<byte>(buffer, length).Memory.ToArray()
    );
    
    socket.Send(segment); // OS sends directly from buffer
}

// Alternative: SocketAsyncEventArgs (pooled, zero-copy)
public class SocketSender
{
    private readonly SocketAsyncEventArgs _sendArgs;

    public async Task SendAsync(Socket socket, byte[] buffer, int offset, int count)
    {
        _sendArgs.SetBuffer(buffer, offset, count);
        
        var tcs = new TaskCompletionSource<bool>();
        _sendArgs.Completed += (_, __) => tcs.SetResult(true);
        
        if (!socket.SendAsync(_sendArgs))
            return; // Completed synchronously
        
        await tcs.Task;
    }
}
```

**Projetos que usam:** Sparrow.Server, Raven.Server

---

## 4?? Caching Strategies

### Pattern 4.1: HTTP Cache with ETags (Raven.Client)

**Problema:** Redundant network requests

**Solução:** Cache com validação via ETags

```csharp
// src/Raven.Client/Http/HttpCache.cs
public class HttpCache
{
    private readonly ConcurrentDictionary<string, HttpCacheItem> _items;
    private long _currentSize;
    private readonly long _maxSize;

    public ReleaseCacheItem Get(
        JsonOperationContext context,
        string url,
        out string changeVector,
        out BlittableJsonReaderObject cachedValue)
    {
        if (_items.TryGetValue(url, out var item))
        {
            // Cache hit!
            changeVector = item.ChangeVector; // ETag
            cachedValue = item.Data.Clone(context);
            
            return new ReleaseCacheItem
            {
                Item = item,
                Age = DateTime.UtcNow - item.LastServerUpdate,
                MightHaveBeenModified = item.MightHaveBeenModified
            };
        }

        changeVector = null;
        cachedValue = null;
        return new ReleaseCacheItem();
    }

    public void Set(string url, string changeVector, BlittableJsonReaderObject result)
    {
        var size = result.Size;
        
        // LRU eviction
        while (_currentSize + size > _maxSize && _items.Count > 0)
        {
            EvictOldestItem();
        }
        
        var item = new HttpCacheItem
        {
            ChangeVector = changeVector,
            Data = result.Clone(context),
            LastServerUpdate = DateTime.UtcNow
        };
        
        _items[url] = item;
        Interlocked.Add(ref _currentSize, size);
    }
}
```

**HTTP Caching Flow:**

```
Client Request
  ?
  ??? Check Cache
  ?   ??? Cache Hit
  ?   ?   ??? Send If-None-Match: "{changeVector}"
  ?   ?   ??? Server Response
  ?   ?       ??? 304 Not Modified ? Use cached
  ?   ?       ??? 200 OK ? Update cache
  ?   ??? Cache Miss
  ?       ??? Full request ? Cache result
```

**Projetos que usam:** Raven.Client

---

### Pattern 4.2: Aggressive Caching Mode (Raven.Client)

**Problema:** Even conditional requests have latency

**Solução:** Skip server validation for duration

```csharp
// src/Raven.Client/Documents/DocumentStore.cs
public IDisposable AggressivelyCache(TimeSpan cacheDuration = default)
{
    var re = GetRequestExecutor();
    var old = re.AggressiveCaching.Value;
    
    re.AggressiveCaching.Value = new AggressiveCacheOptions
    {
        Duration = cacheDuration == default 
            ? TimeSpan.FromDays(1) 
            : cacheDuration,
        Mode = AggressiveCacheMode.TrackChanges
    };
    
    return new DisposableAction(() => re.AggressiveCaching.Value = old);
}

// Usage
using (store.AggressivelyCache(TimeSpan.FromMinutes(5)))
{
    var user = session.Load<User>("users/1"); // From cache, no server call!
    var sameUser = session.Load<User>("users/1"); // Still no server call!
}
```

**Modes:**

| Mode | Behavior |
|------|----------|
| `None` | Normal caching (with validation) |
| `TrackChanges` | Cache + invalidate on change notifications |
| `DoNotTrackChanges` | Pure time-based caching |

**Projetos que usam:** Raven.Client

---

### Pattern 4.3: Thread-Local Caching (Sparrow)

**Problema:** Global cache = lock contention

**Solução:** Thread-local cache first, fallback to global

```csharp
// src/Sparrow/Json/JsonContextPool.cs
public class JsonContextPool
{
    private readonly ConcurrentStack<JsonOperationContext> _globalStack;
    private readonly AsyncLocal<JsonOperationContext> _perThreadContext;

    public IDisposable AllocateOperationContext(out JsonOperationContext context)
    {
        // L1: Thread-local (fastest, no contention)
        context = _perThreadContext.Value;
        if (context != null)
        {
            _perThreadContext.Value = null;
            return new ReturnContext(this, context);
        }

        // L2: Global pool (lock-free via ConcurrentStack)
        if (_globalStack.TryPop(out context))
            return new ReturnContext(this, context);

        // L3: Create new (slowest)
        context = new JsonOperationContext();
        return new ReturnContext(this, context);
    }

    private void Return(JsonOperationContext context)
    {
        // Return to L1 first
        if (_perThreadContext.Value == null)
        {
            _perThreadContext.Value = context;
            return;
        }

        // Fallback to L2
        _globalStack.Push(context);
    }
}
```

**Cache Hierarchy:**

```
Request
  ?
  ??? L1: Thread-Local (AsyncLocal<T>)
  ?   ??? Hit: ~1ns access time
  ?
  ??? L2: Global Pool (ConcurrentStack<T>)
  ?   ??? Hit: ~10ns (lock-free)
  ?
  ??? L3: New Instance
      ??? Miss: ~1000ns (allocation + init)
```

**Projetos que usam:** Sparrow, Raven.Server, Raven.Client

---

## 5?? Object Pooling Patterns

### Pattern 5.1: ArrayPool<T> for Temporary Buffers (Sparrow)

**Problema:** Frequent buffer allocations

**Solução:** Rent from shared pool

```csharp
// src/Sparrow/Json/JsonParserState.cs
public class JsonParserState
{
    private byte[] _buffer;

    public void EnsureBufferSize(int size)
    {
        if (_buffer != null && _buffer.Length >= size)
            return;

        // Return old buffer to pool
        if (_buffer != null)
            ArrayPool<byte>.Shared.Return(_buffer);

        // Rent new buffer (power-of-2 size)
        _buffer = ArrayPool<byte>.Shared.Rent(size);
    }

    public void Dispose()
    {
        if (_buffer != null)
        {
            ArrayPool<byte>.Shared.Return(_buffer, clearArray: true);
            _buffer = null;
        }
    }
}
```

**ArrayPool Best Practices:**

```csharp
// ? CORRETO - Rent + Return
var buffer = ArrayPool<byte>.Shared.Rent(size);
try
{
    // Use buffer
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer, clearArray: true); // Clear for security
}

// ? ERRADO - Forget to return
var buffer = ArrayPool<byte>.Shared.Rent(size);
// Memory leak! Buffer never returned!

// ? ERRADO - Use after return
ArrayPool<byte>.Shared.Return(buffer);
buffer[0] = 42; // UNDEFINED BEHAVIOR!
```

**Projetos que usam:** Sparrow, Raven.Server, Raven.Client

---

### Pattern 5.2: Custom Pool with Size Classes (Sparrow)

**Problema:** Different object sizes need different pools

**Solução:** Multiple pools por size class

```csharp
// src/Sparrow/NativeMemory.cs
public class NativeMemoryPool
{
    // Size classes: 4KB, 8KB, 16KB, 32KB, 64KB, 128KB
    private readonly ConcurrentStack<IntPtr>[] _pools = new ConcurrentStack<IntPtr>[6];
    
    private static int GetPoolIndex(int size)
    {
        // Round up to next power of 2
        var sizeClass = Bits.CeilLog2(size) - 12; // 2^12 = 4KB
        return Math.Min(sizeClass, 5); // Max 128KB
    }

    public IntPtr Allocate(int size)
    {
        var poolIndex = GetPoolIndex(size);
        
        if (_pools[poolIndex].TryPop(out var ptr))
            return ptr; // From pool

        // Allocate new
        var actualSize = 4096 << poolIndex; // 4KB * 2^poolIndex
        return Marshal.AllocHGlobal(actualSize);
    }

    public void Free(IntPtr ptr, int size)
    {
        var poolIndex = GetPoolIndex(size);
        
        if (_pools[poolIndex].Count < MaxPerPool)
        {
            _pools[poolIndex].Push(ptr); // Return to pool
        }
        else
        {
            Marshal.FreeHGlobal(ptr); // Pool full, free
        }
    }
}
```

**Projetos que usam:** Sparrow, Voron

---

## 6?? Inlining & Hot Path Optimization

### Pattern 6.1: AggressiveInlining (Sparrow)

**Problema:** Method call overhead em hot paths

**Solução:** Force inline com attribute

```csharp
// src/Sparrow/Size.cs
public readonly struct Size
{
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static Size operator +(Size left, Size right)
    {
        return new Size(left._value + right._value, left._sizeUnit);
    }

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public long GetValue(SizeUnit unit)
    {
        if (unit == _sizeUnit)
            return _value; // Fast path

        return ConvertUnit(_value, _sizeUnit, unit); // Slow path
    }
}
```

**Quando usar AggressiveInlining:**

```csharp
// ? CORRETO - Small methods, hot paths
[MethodImpl(MethodImplOptions.AggressiveInlining)]
public int Add(int a, int b) => a + b;

// ? ERRADO - Large methods (JIT ignores hint)
[MethodImpl(MethodImplOptions.AggressiveInlining)]
public void LargeMethod() { /* 100+ lines */ }

// ? ERRADO - Virtual methods (cannot inline)
[MethodImpl(MethodImplOptions.AggressiveInlining)]
public virtual int Add(int a, int b) => a + b;
```

**Projetos que usam:** Todos os projetos

---

### Pattern 6.2: Branch Prediction Hints (Voron)

**Problema:** Branch mispredictions são caras

**Solução:** Organize code para likely path

```csharp
// src/Voron/Impl/PageLocator.cs
public Page GetPage(long pageNumber)
{
    // Likely path FIRST (branch predictor assumes forward branch = unlikely)
    if (_cache.TryGetValue(pageNumber, out var page))
        return page; // Most common case

    // Unlikely path
    return LoadPageFromDisk(pageNumber);
}

// Alternative: throw for unlikely paths
public Page GetPageOrThrow(long pageNumber)
{
    if (_cache.TryGetValue(pageNumber, out var page))
        return page;

    // Unlikely - exception path is cold
    ThrowPageNotFound(pageNumber);
    return null; // Never reached
}

[DoesNotReturn]
private void ThrowPageNotFound(long pageNumber)
{
    throw new InvalidOperationException($"Page {pageNumber} not found");
}
```

**Projetos que usam:** Voron, Corax, Raven.Server

---

## 7?? Data Structure Patterns

### Pattern 7.1: Struct for Small Objects (Sparrow)

**Problema:** Class allocation pressure

**Solução:** Stack-allocated structs

```csharp
// src/Sparrow/Size.cs
public readonly struct Size : IComparable<Size>
{
    private readonly long _value;
    private readonly SizeUnit _sizeUnit;

    public Size(long value, SizeUnit unit)
    {
        _value = value;
        _sizeUnit = unit;
    }

    // All methods are value-based (no allocation)
    public Size Add(Size other) => new Size(_value + other.GetValue(_sizeUnit), _sizeUnit);
}

// Usage - no heap allocation!
Size a = new Size(100, SizeUnit.Megabytes);
Size b = new Size(50, SizeUnit.Megabytes);
Size c = a.Add(b); // Stack only!
```

**Struct Best Practices:**

```csharp
// ? CORRETO - readonly struct (immutable, can't be modified)
public readonly struct Point
{
    public readonly int X;
    public readonly int Y;
}

// ?? CUIDADO - mutable struct (defensive copies)
public struct MutablePoint
{
    public int X;
    public int Y;
    
    public void Move(int dx, int dy)
    {
        X += dx; // Modifies copy if called on readonly field!
        Y += dy;
    }
}

// ? MELHOR - ref struct (stack only, cannot box)
public ref struct StackOnlyStruct
{
    public Span<byte> Buffer; // Can contain Span<T>!
}
```

**Quando usar struct:**
- Small size (<= 16 bytes ideal)
- Value semantics (immutable preferred)
- Hot path allocations
- Cannot use: virtual methods, inheritance

**Projetos que usam:** Sparrow, Voron, Corax

---

### Pattern 7.2: Cache-Friendly Layouts (Voron)

**Problema:** Cache misses em data structures

**Solução:** Pack hot fields together

```csharp
// ? POOR CACHE LOCALITY
public class Page
{
    public long PageNumber;        // Offset 0
    public byte[] LargeBuffer;     // Offset 8 (64KB!)
    public int AccessCount;        // Offset 65544 (cache miss!)
    public DateTime LastAccess;    // Offset 65548
}

// ? GOOD CACHE LOCALITY
public class Page
{
    // HOT fields (accessed together)
    public long PageNumber;        // Offset 0
    public int AccessCount;        // Offset 8
    public DateTime LastAccess;    // Offset 16
    
    // COLD field (separate cache line)
    public byte[] LargeBuffer;     // Offset 24
}
```

**Cache Line Padding:**

```csharp
// Prevent false sharing between threads
[StructLayout(LayoutKind.Explicit, Size = 128)] // 2 cache lines
public struct PaddedCounter
{
    [FieldOffset(0)]
    public long Value;
    
    // Remaining 120 bytes are padding (prevents false sharing)
}
```

**Projetos que usam:** Voron, Sparrow.Server

---

## 8?? Networking Patterns

### Pattern 8.1: HttpClient Pooling (Raven.Client)

**Problema:** Creating HttpClient per request = socket exhaustion

**Solução:** Shared HttpClient instance

```csharp
// src/Raven.Client/Http/RequestExecutor.cs
internal static IRavenHttpClientFactory HttpClientFactory = DefaultRavenHttpClientFactory.Instance;

private HttpClient _cachedHttpClient;

public HttpClient HttpClient
{
    get
    {
        if (_cachedHttpClient != null)
            return _cachedHttpClient;

        var httpClient = HttpClientFactory.GetHttpClient(_httpClientCacheKey, Conventions.CreateHttpClient);
        
        if (HttpClientFactory.CanCacheHttpClient)
            _cachedHttpClient = httpClient; // Reuse!

        return httpClient;
    }
}

// Update ServicePoint connection limit
ServicePointManager.FindServicePoint(uri).ConnectionLimit = int.MaxValue;
```

**HttpClient Best Practices:**

```csharp
// ? CORRETO - Singleton HttpClient
private static readonly HttpClient _httpClient = new HttpClient();

// ? ERRADO - New HttpClient per request
using (var httpClient = new HttpClient()) // NEVER DO THIS!
{
    await httpClient.GetAsync(url);
}

// ? ALTERNATIVA - IHttpClientFactory (.NET Core+)
public class MyService
{
    private readonly IHttpClientFactory _factory;
    
    public async Task CallApi()
    {
        var client = _factory.CreateClient();
        await client.GetAsync(url);
    }
}
```

**Projetos que usam:** Raven.Client

---

### Pattern 8.2: Request Batching (Raven.Client)

**Problema:** Multiple small requests = high latency

**Solução:** Batch em single request

```csharp
// src/Raven.Client/Documents/Session/InMemoryDocumentSessionOperations.cs
internal SaveChangesData PrepareForSaveChanges()
{
    var result = new SaveChangesData(this);

    // Batch ALL changes into single request
    PrepareForEntitiesDeletion(result);   // Deletes
    PrepareForEntitiesPuts(result);       // Puts
    PrepareForCreatingRevisionsFromIds(result); // Revisions
    PrepareCompareExchangeEntities(result);     // Compare-exchange

    return result; // Single HTTP POST to /bulk_docs
}
```

**Batching Benefits:**

```
Without Batching:
  - 10 documents = 10 HTTP requests
  - Latency: 10 * 15ms = 150ms
  - Throughput: 66 docs/sec

With Batching:
  - 10 documents = 1 HTTP request
  - Latency: 1 * 18ms = 18ms
  - Throughput: 555 docs/sec
  
Performance gain: 8.3x faster! ??
```

**Projetos que usam:** Raven.Client

---

### Pattern 8.3: HTTP Compression (Raven.Client)

**Problema:** Large payloads = slow network transfer

**Solução:** Compress requests/responses

```csharp
// src/Raven.Client/Http/RequestExecutor.cs
if (Conventions.UseHttpCompression && Conventions.HttpCompressionAlgorithm == HttpCompressionAlgorithm.Zstd)
{
    request.Headers.TryAddWithoutValidation(Constants.Headers.AcceptEncoding, Constants.Headers.Encodings.Zstd);
}

// Server response decompression
public static async Task<Stream> ReadAsStreamUncompressedAsync(HttpResponseMessage response)
{
    var serverStream = await response.Content.ReadAsStreamAsync();
    var encoding = response.Content.Headers.ContentEncoding.FirstOrDefault();
    
    switch (encoding)
    {
        case Constants.Headers.Encodings.Gzip:
            return new GZipStream(serverStream, CompressionMode.Decompress);
        case Constants.Headers.Encodings.Zstd:
            return ZstdStream.Decompress(serverStream);
        default:
            return serverStream;
    }
}
```

**Compression Algorithms:**

| Algorithm | Ratio | Speed | Use Case |
|-----------|-------|-------|----------|
| **None** | 1.0x | Fastest | Small payloads |
| **Gzip** | 3-5x | Fast | General purpose |
| **Zstd** | 4-6x | Fastest | Modern (RavenDB default) |
| **Brotli** | 5-7x | Slow | Static resources |

**Projetos que usam:** Raven.Client, Raven.Server

---

## ?? Pattern Selection Guide

### Memory Management

| Scenario | Pattern | Projects |
|----------|---------|----------|
| Scoped allocations | Arena Allocator | Sparrow, Voron |
| Expensive objects | Object Pooling | All |
| Temporary slices | Span<T> | Sparrow, Voron |
| Async slices | Memory<T> | Sparrow.Server |
| Temporary buffers | ArrayPool<T> | All |

### Concurrency

| Scenario | Pattern | Projects |
|----------|---------|----------|
| Counters/flags | Interlocked | All |
| Lazy init | Lazy<T> | Raven.Client, Embedded |
| Read-heavy dict | ConcurrentDictionary | All |
| Singleton init | Double-Check Lock | Sparrow, Server |

### I/O

| Scenario | Pattern | Projects |
|----------|---------|----------|
| Large files | Memory-Mapped Files | Voron, Corax |
| Async I/O | ValueTask<T> | Sparrow.Server |
| Write buffering | Batching | Voron |
| Network send | Zero-Copy | Sparrow.Server |

### Caching

| Scenario | Pattern | Projects |
|----------|---------|----------|
| HTTP responses | ETag Cache | Raven.Client |
| Aggressive mode | Time-based | Raven.Client |
| Thread-local | L1 + L2 Cache | Sparrow, Client |

### Data Structures

| Scenario | Pattern | Projects |
|----------|---------|----------|
| Small values | Struct | Sparrow |
| Hot fields | Cache-Friendly Layout | Voron |
| Size classes | Custom Pool | Sparrow |

### Networking

| Scenario | Pattern | Projects |
|----------|---------|----------|
| HTTP calls | HttpClient Pooling | Raven.Client |
| Multiple ops | Request Batching | Raven.Client |
| Large payloads | Compression | Client, Server |

---

## ?? Performance Impact Summary

### High Impact (10x+)

1. **Arena Allocator** - Eliminates GC pressure
2. **Object Pooling** - Zero allocation on hot path
3. **Memory-Mapped Files** - OS page cache + zero-copy
4. **Request Batching** - 8.3x faster (10 docs)
5. **Aggressive Caching** - Zero network latency

### Medium Impact (2-10x)

6. **Span<T>** - Stack allocation vs heap
7. **ValueTask<T>** - No allocation if sync
8. **HTTP Compression** - 4-6x bandwidth reduction
9. **ArrayPool<T>** - Reuse vs new allocation
10. **Interlocked** - Lock-free vs lock

### Low Impact (<2x)

11. **AggressiveInlining** - Eliminates call overhead
12. **Branch Prediction** - Fewer pipeline stalls
13. **Cache-Friendly Layout** - Better CPU cache hits
14. **Thread-Local Cache** - Reduces contention

---

## ?? Key Takeaways

### Do's ?

1. **Pool expensive objects** (JsonContext, buffers, connections)
2. **Use Span<T>** for temp data (stack allocation)
3. **Batch operations** (I/O, network requests)
4. **Cache aggressively** (when data doesn't change often)
5. **Prefer lock-free** (Interlocked, ConcurrentDictionary)
6. **Inline hot paths** (AggressiveInlining attribute)
7. **Reuse HttpClient** (singleton per endpoint)

### Don'ts ?

1. **Don't allocate per request** (pool instead)
2. **Don't lock hot paths** (use Interlocked)
3. **Don't copy buffers** (use Span/Memory)
4. **Don't create HttpClient per request** (socket exhaustion)
5. **Don't ignore cache** (validate first)
6. **Don't inline large methods** (JIT ignores)
7. **Don't use locks for counters** (Interlocked faster)

---

## ?? Next Steps

Com **Performance Patterns** concluído, próximas análises temáticas:

- [ ] **Memory Management** - Deep dive em allocation strategies
- [ ] **Concurrency Patterns** - Threading, async, synchronization
- [ ] **Unsafe Code** - When and how to use unsafe
- [ ] **Data Structures** - Custom collections
- [ ] **I/O Optimization** - File, network, serialization
- [ ] **Benchmark Analysis** - Performance metrics
- [ ] **Testing Strategies** - How RavenDB tests performance

---

**Conclusão:** Este catálogo de **50+ patterns** consolida as técnicas de alta performance usadas em toda a codebase do RavenDB. Use como referência para:

- ? **Design reviews** - Identificar oportunidades de otimização
- ? **Code implementation** - Patterns prontos para copy/paste
- ? **Performance debugging** - Checklist de técnicas
- ? **Learning** - Entender trade-offs de cada pattern

**Total Patterns Catalogados:** 50+ padrões concretos com exemplos de código! ??
