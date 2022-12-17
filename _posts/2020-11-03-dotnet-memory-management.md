---
layout: post
title: ".NET Core 記憶體管理與效能優化"
date: 2020-11-03 15:00:00 +0800
categories: [框架, .NET]
tags: [.NET Core, Performance, Memory, GC]
---

這週研究 .NET Core 的記憶體管理機制、垃圾回收原理，以及如何診斷和優化記憶體使用。

> 本文環境：**.NET Core 3.1 LTS** + **C# 9.0**

## 垃圾回收機制

### GC 世代

```csharp
public class GCGenerations
{
    public void DemonstrateGenerations()
    {
        var obj = new object();
        
        // 取得物件所在的世代
        int generation = GC.GetGeneration(obj);
        Console.WriteLine($"物件在世代 {generation}");

        // 觸發垃圾回收
        GC.Collect(0); // 只回收 Gen 0
        generation = GC.GetGeneration(obj);
        Console.WriteLine($"GC 後物件在世代 {generation}");

        // 取得各世代的回收次數
        Console.WriteLine($"Gen 0: {GC.CollectionCount(0)} 次");
        Console.WriteLine($"Gen 1: {GC.CollectionCount(1)} 次");
        Console.WriteLine($"Gen 2: {GC.CollectionCount(2)} 次");
    }

    public void MonitorMemory()
    {
        var before = GC.GetTotalMemory(false);
        
        // 執行一些操作
        var list = new List<byte[]>();
        for (int i = 0; i < 1000; i++)
        {
            list.Add(new byte[1024]);
        }

        var after = GC.GetTotalMemory(false);
        Console.WriteLine($"記憶體增加: {(after - before) / 1024} KB");

        // 強制垃圾回收並等待
        GC.Collect();
        GC.WaitForPendingFinalizers();
        GC.Collect();

        var final = GC.GetTotalMemory(true);
        Console.WriteLine($"GC 後記憶體: {final / 1024} KB");
    }
}
```

### GC 模式

```csharp
// 設定 GC 模式（runtimeconfig.json 或 csproj）
// Workstation GC vs Server GC

// runtimeconfig.json
{
  "runtimeOptions": {
    "configProperties": {
      "System.GC.Server": true,
      "System.GC.Concurrent": true,
      "System.GC.RetainVM": false
    }
  }
}
```

```xml
<!-- csproj -->
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
  <ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>
  <RetainVMGarbageCollection>false</RetainVMGarbageCollection>
</PropertyGroup>
```

## IDisposable 模式

### 正確實作 Dispose

```csharp
public class ResourceManager : IDisposable
{
    private bool _disposed = false;
    private IntPtr _unmanagedResource;
    private FileStream _managedResource;

    public ResourceManager()
    {
        _unmanagedResource = // 分配非託管資源
        _managedResource = new FileStream("data.txt", FileMode.Open);
    }

    // 實作 IDisposable
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    // 受保護的 Dispose 方法
    protected virtual void Dispose(bool disposing)
    {
        if (_disposed)
        {
            return;
        }

        if (disposing)
        {
            // 釋放託管資源
            _managedResource?.Dispose();
        }

        // 釋放非託管資源
        if (_unmanagedResource != IntPtr.Zero)
        {
            // 釋放非託管資源的程式碼
            _unmanagedResource = IntPtr.Zero;
        }

        _disposed = true;
    }

    // Finalizer
    ~ResourceManager()
    {
        Dispose(false);
    }

    // 檢查是否已釋放
    private void ThrowIfDisposed()
    {
        if (_disposed)
        {
            throw new ObjectDisposedException(GetType().Name);
        }
    }

    public void DoSomething()
    {
        ThrowIfDisposed();
        // 執行操作
    }
}
```

### 非同步 Dispose

```csharp
public class AsyncResourceManager : IAsyncDisposable, IDisposable
{
    private readonly HttpClient _httpClient;
    private readonly Stream _stream;
    private bool _disposed;

    public async ValueTask DisposeAsync()
    {
        if (_disposed)
        {
            return;
        }

        await DisposeAsyncCore().ConfigureAwait(false);

        Dispose(false);
        GC.SuppressFinalize(this);
    }

    protected virtual async ValueTask DisposeAsyncCore()
    {
        if (_stream is not null)
        {
            await _stream.DisposeAsync().ConfigureAwait(false);
        }

        _httpClient?.Dispose();
    }

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed)
        {
            return;
        }

        if (disposing)
        {
            _stream?.Dispose();
            _httpClient?.Dispose();
        }

        _disposed = true;
    }
}

// 使用
public async Task UseResourceAsync()
{
    await using var resource = new AsyncResourceManager();
    // 使用資源
}
```

## Span<T> 和 Memory<T>

### 使用 Span 減少配置

```csharp
public class SpanExample
{
    // 傳統方式 - 產生新字串
    public string SubstringOld(string input, int start, int length)
    {
        return input.Substring(start, length);
    }

    // 使用 Span - 不產生新字串
    public ReadOnlySpan<char> SubstringNew(string input, int start, int length)
    {
        return input.AsSpan(start, length);
    }

    // 解析整數
    public int ParseInteger(string input)
    {
        var span = input.AsSpan().Trim();
        return int.Parse(span);
    }

    // 操作陣列片段
    public void ProcessArraySegment(int[] array)
    {
        // 傳統方式 - 建立新陣列
        var segment = new int[10];
        Array.Copy(array, 0, segment, 0, 10);

        // 使用 Span - 不建立新陣列
        var span = array.AsSpan(0, 10);
        ProcessSpan(span);
    }

    private void ProcessSpan(Span<int> span)
    {
        for (int i = 0; i < span.Length; i++)
        {
            span[i] *= 2;
        }
    }

    // Stack 配置
    public void StackAllocation()
    {
        Span<byte> buffer = stackalloc byte[256];
        
        for (int i = 0; i < buffer.Length; i++)
        {
            buffer[i] = (byte)i;
        }

        ProcessBuffer(buffer);
    }

    private void ProcessBuffer(Span<byte> buffer)
    {
        // 處理緩衝區
    }
}
```

### Memory<T> 用於非同步

```csharp
public class MemoryExample
{
    // Span 不能在 async 方法中使用
    public async Task<int> ReadDataAsync(Stream stream)
    {
        // 使用 Memory 代替 Span
        Memory<byte> buffer = new byte[4096];
        
        int bytesRead = await stream.ReadAsync(buffer);
        
        // 轉換為 Span 進行處理
        Span<byte> span = buffer.Span.Slice(0, bytesRead);
        return ProcessData(span);
    }

    private int ProcessData(Span<byte> data)
    {
        // 處理資料
        return data.Length;
    }

    // 使用 ArrayPool 減少配置
    public async Task ProcessLargeDataAsync(Stream stream)
    {
        var buffer = ArrayPool<byte>.Shared.Rent(4096);
        try
        {
            var memory = buffer.AsMemory(0, 4096);
            int bytesRead = await stream.ReadAsync(memory);
            
            // 處理資料
            ProcessData(buffer.AsSpan(0, bytesRead));
        }
        finally
        {
            ArrayPool<byte>.Shared.Return(buffer);
        }
    }
}
```

## ArrayPool

### 物件池減少配置

```csharp
public class ArrayPoolExample
{
    public void ProcessData()
    {
        // 傳統方式 - 每次都配置新陣列
        var buffer = new byte[1024];
        // 使用 buffer
    }

    public void ProcessDataWithPool()
    {
        // 從池中租用
        var buffer = ArrayPool<byte>.Shared.Rent(1024);
        try
        {
            // 使用 buffer
            Array.Clear(buffer, 0, buffer.Length);
            
            // 處理資料
        }
        finally
        {
            // 歸還到池中
            ArrayPool<byte>.Shared.Return(buffer);
        }
    }

    // 自訂池大小
    public void CreateCustomPool()
    {
        var pool = ArrayPool<byte>.Create(
            maxArrayLength: 1024 * 1024,
            maxArraysPerBucket: 50);

        var buffer = pool.Rent(1024);
        try
        {
            // 使用 buffer
        }
        finally
        {
            pool.Return(buffer, clearArray: true);
        }
    }
}
```

### 實作自訂物件池

```csharp
public class ObjectPool<T> where T : class
{
    private readonly ConcurrentBag<T> _objects;
    private readonly Func<T> _objectGenerator;
    private readonly Action<T> _resetAction;

    public ObjectPool(Func<T> objectGenerator, Action<T> resetAction = null)
    {
        _objects = new ConcurrentBag<T>();
        _objectGenerator = objectGenerator ?? throw new ArgumentNullException(nameof(objectGenerator));
        _resetAction = resetAction;
    }

    public T Rent()
    {
        return _objects.TryTake(out T item) ? item : _objectGenerator();
    }

    public void Return(T item)
    {
        _resetAction?.Invoke(item);
        _objects.Add(item);
    }
}

// 使用
public class StringBuilderPool
{
    private static readonly ObjectPool<StringBuilder> _pool = new ObjectPool<StringBuilder>(
        () => new StringBuilder(256),
        sb => sb.Clear());

    public string BuildString(string[] parts)
    {
        var sb = _pool.Rent();
        try
        {
            foreach (var part in parts)
            {
                sb.Append(part);
            }
            return sb.ToString();
        }
        finally
        {
            _pool.Return(sb);
        }
    }
}
```

## 記憶體診斷

### 使用 dotnet-dump

```bash
# 安裝 dotnet-dump
dotnet tool install -g dotnet-dump

# 建立記憶體傾印
dotnet-dump collect -p <process-id>

# 分析傾印檔
dotnet-dump analyze <dump-file>

# 常用命令
> dumpheap -stat              # 顯示物件統計
> dumpheap -mt <method-table> # 顯示特定類型的物件
> gcroot <object-address>     # 找出物件的根參照
> eeheap -gc                  # 顯示 GC Heap 資訊
```

### 使用 dotnet-counters

```bash
# 安裝 dotnet-counters
dotnet tool install -g dotnet-counters

# 監控效能計數器
dotnet-counters monitor -p <process-id>

# 監控特定計數器
dotnet-counters monitor -p <process-id> \
  --counters System.Runtime[cpu-usage,working-set,gc-heap-size,gen-0-gc-count]
```

### 使用 dotnet-trace

```bash
# 安裝 dotnet-trace
dotnet tool install -g dotnet-trace

# 收集追蹤
dotnet-trace collect -p <process-id> --profile gc-collect

# 轉換為 speedscope 格式
dotnet-trace convert trace.nettrace --format Speedscope
```

## 記憶體洩漏偵測

### 常見記憶體洩漏

```csharp
public class MemoryLeakExamples
{
    // 錯誤 1: 事件未取消訂閱
    public class Publisher
    {
        public event EventHandler DataReceived;
    }

    public class BadSubscriber
    {
        public BadSubscriber(Publisher publisher)
        {
            publisher.DataReceived += OnDataReceived;
            // 忘記取消訂閱
        }

        private void OnDataReceived(object sender, EventArgs e) { }
    }

    public class GoodSubscriber : IDisposable
    {
        private readonly Publisher _publisher;

        public GoodSubscriber(Publisher publisher)
        {
            _publisher = publisher;
            _publisher.DataReceived += OnDataReceived;
        }

        public void Dispose()
        {
            _publisher.DataReceived -= OnDataReceived;
        }

        private void OnDataReceived(object sender, EventArgs e) { }
    }

    // 錯誤 2: 靜態集合持續增長
    public class BadCache
    {
        private static readonly Dictionary<string, byte[]> _cache = new();

        public void AddToCache(string key, byte[] data)
        {
            _cache[key] = data; // 永遠不會被釋放
        }
    }

    public class GoodCache
    {
        private static readonly MemoryCache _cache = new MemoryCache(new MemoryCacheOptions
        {
            SizeLimit = 1024
        });

        public void AddToCache(string key, byte[] data)
        {
            var options = new MemoryCacheEntryOptions
            {
                Size = data.Length,
                SlidingExpiration = TimeSpan.FromMinutes(10)
            };

            _cache.Set(key, data, options);
        }
    }

    // 錯誤 3: Timer 未釋放
    public class BadTimer
    {
        private System.Threading.Timer _timer;

        public void Start()
        {
            _timer = new System.Threading.Timer(
                _ => Console.WriteLine("Tick"),
                null,
                TimeSpan.Zero,
                TimeSpan.FromSeconds(1));
            // 忘記 Dispose
        }
    }

    public class GoodTimer : IDisposable
    {
        private System.Threading.Timer _timer;

        public void Start()
        {
            _timer = new System.Threading.Timer(
                _ => Console.WriteLine("Tick"),
                null,
                TimeSpan.Zero,
                TimeSpan.FromSeconds(1));
        }

        public void Dispose()
        {
            _timer?.Dispose();
        }
    }
}
```

### 使用 WeakReference

```csharp
public class WeakReferenceExample
{
    // 快取但允許 GC 回收
    public class WeakCache<TKey, TValue> where TValue : class
    {
        private readonly Dictionary<TKey, WeakReference<TValue>> _cache = new();
        private readonly object _lock = new();

        public bool TryGetValue(TKey key, out TValue value)
        {
            lock (_lock)
            {
                if (_cache.TryGetValue(key, out var weakRef))
                {
                    if (weakRef.TryGetTarget(out value))
                    {
                        return true;
                    }

                    // 物件已被回收，移除項目
                    _cache.Remove(key);
                }

                value = null;
                return false;
            }
        }

        public void Add(TKey key, TValue value)
        {
            lock (_lock)
            {
                _cache[key] = new WeakReference<TValue>(value);
            }
        }

        public void Clean()
        {
            lock (_lock)
            {
                var keysToRemove = _cache
                    .Where(kvp => !kvp.Value.TryGetTarget(out _))
                    .Select(kvp => kvp.Key)
                    .ToList();

                foreach (var key in keysToRemove)
                {
                    _cache.Remove(key);
                }
            }
        }
    }
}
```

## 效能最佳化技巧

### 字串處理優化

```csharp
public class StringOptimization
{
    // 錯誤：大量字串串接
    public string ConcatenateBad(string[] parts)
    {
        string result = "";
        foreach (var part in parts)
        {
            result += part; // 每次都建立新字串
        }
        return result;
    }

    // 較好：使用 StringBuilder
    public string ConcatenateGood(string[] parts)
    {
        var sb = new StringBuilder();
        foreach (var part in parts)
        {
            sb.Append(part);
        }
        return sb.ToString();
    }

    // 最好：使用 string.Join
    public string ConcatenateBest(string[] parts)
    {
        return string.Join("", parts);
    }

    // 使用 Span 避免配置
    public bool StartsWithPrefix(string input, string prefix)
    {
        return input.AsSpan().StartsWith(prefix.AsSpan(), StringComparison.Ordinal);
    }

    // 字串插值優化
    public string FormatBad(int id, string name)
    {
        return $"ID: {id}, Name: {name}"; // 配置
    }

    public string FormatGood(int id, string name)
    {
        var handler = new DefaultInterpolatedStringHandler(12, 2);
        handler.AppendLiteral("ID: ");
        handler.AppendFormatted(id);
        handler.AppendLiteral(", Name: ");
        handler.AppendFormatted(name);
        return handler.ToStringAndClear();
    }
}
```

### 集合優化

```csharp
public class CollectionOptimization
{
    // 預先設定容量
    public List<int> CreateListBad()
    {
        var list = new List<int>(); // 預設容量 4
        for (int i = 0; i < 1000; i++)
        {
            list.Add(i); // 多次重新配置
        }
        return list;
    }

    public List<int> CreateListGood()
    {
        var list = new List<int>(1000); // 預設容量 1000
        for (int i = 0; i < 1000; i++)
        {
            list.Add(i); // 不需要重新配置
        }
        return list;
    }

    // 選擇正確的集合類型
    public void UseAppropriateCollection()
    {
        // 需要快速查找：Dictionary
        var dict = new Dictionary<int, string>();

        // 需要唯一值：HashSet
        var set = new HashSet<int>();

        // 需要順序：List
        var list = new List<int>();

        // 需要 FIFO：Queue
        var queue = new Queue<int>();

        // 需要 LIFO：Stack
        var stack = new Stack<int>();

        // 需要排序：SortedDictionary
        var sorted = new SortedDictionary<int, string>();
    }
}
```

### 避免裝箱

```csharp
public class BoxingOptimization
{
    // 錯誤：裝箱
    public void AddToListBad(object list, int value)
    {
        ((List<object>)list).Add(value); // 裝箱
    }

    // 正確：使用泛型
    public void AddToListGood<T>(List<T> list, T value)
    {
        list.Add(value); // 不裝箱
    }

    // 錯誤：ToString() 造成裝箱
    public string FormatBad(int value)
    {
        return string.Format("{0}", value); // 裝箱
    }

    // 正確：使用字串插值或 FormattableString
    public string FormatGood(int value)
    {
        return $"{value}"; // 不裝箱（編譯器優化）
    }
}
```

## 小結

.NET Core 記憶體管理：
- 理解 GC 世代和回收機制
- 正確實作 IDisposable 和 IAsyncDisposable
- 使用 Span<T> 和 Memory<T> 減少配置
- 使用 ArrayPool 和物件池重用記憶體
- 使用診斷工具找出記憶體問題

效能優化重點：
- 避免不必要的物件配置
- 選擇適當的集合類型
- 避免裝箱和拆箱
- 預先設定集合容量
- 使用 Span<T> 處理字串和陣列

下週將探討 .NET Core 的 CI/CD 部署策略。
