# Space Development Coding Patterns and Best Practices

This document defines the **coding patterns, architectural conventions, and best practices** used in Space Development projects.

---

## 1. Argument Validation

### 1.1 Check vs Assertion

Two validation classes exist with identical APIs but different purposes:

| Class | Behavior | Use Case |
|-------|----------|----------|
| `ArgumentException, ArgumentNullException, ArgumentOutOfRangeException` | Always executes | Public API boundaries, user input |
| `Assertion` | DEBUG builds only | Internal invariants, performance-critical paths |

```csharp
public void ProcessData(string name, int count, object internalState)
{
    // Public parameters - always validate
    ArgumentException.ThrowIfNullOrEmpty(name, nameof(name));
    ArgumentOutOfRangeException.ThrowIfNegative(count, nameof(count));
    
    // Internal state - debug-only validation
    Assertion.ThrowIfNull(internalState, nameof(internalState));
}
```

### 1.2 Available Assertion Methods

```csharp
// Null checks
Assertion.ThrowIfNull(value, nameof(value));
Assertion.ThrowIfNullOrEmpty(stringValue, nameof(stringValue));

// Numeric bounds
Assertion.ThrowIfNegative(value, nameof(value));
Assertion.ThrowIfNegativeOrZero(value, nameof(value));
Assertion.ThrowIfLessThan(value, limit, nameof(value));
Assertion.ThrowIfGreaterThan(value, limit, nameof(value));
Assertion.ThrowIfGreaterThanOrEqual(value, limit, nameof(value));

// Equality
Assertion.ThrowIfEqual(value1, value2, nameof(value1));
Assertion.ThrowIfNotEqual(value1, value2, nameof(value1));
```

### 1.3 Validation Placement

```csharp
public class CircularList<T>
{
    public CircularList(int capacity)
    {
        // Validate at entry point
        Assertion.ThrowIfLessThan(capacity, 2, nameof(capacity));
        
        buffer = new T[capacity];
    }
    
    public T this[int index]
    {
        get
        {
            // Validate accessor parameters
            ArgumentOutOfRangeException.ThrowIfLessThan(index, 0, nameof(index));
            ArgumentOutOfRangeException.ThrowIfGreaterThanOrEqual(index, Count, nameof(index));
            
            return buffer[(head + index) % Capacity];
        }
    }
}
```

---

## 2. Base Classes

### 2.1 DisposableBase and SafeDisposable

**DisposableBase** class is an abstract base classes for disposable objects with the standard IDisposable implementation:

```csharp
public abstract class DisposableBase : IDisposable
{
    ~DisposableBase()
    {
        Dispose(false);
    }

    public void Dispose()
    {
        if (isDisposed)
        {
            throw new ObjectDisposedException(GetType().Name);
        }

        Dispose(true);
        isDisposed = true;

        GC.SuppressFinalize(this);
    }

    protected abstract void Dispose(bool disposing);
}
```

**SafeDisposableBase** class is the thread-safe abstract base classes implementation for disposable objects. It also provides a protected SyncRoot object for objectwise synchronization purposes.


**Usage:**
```csharp
public class MyResource : DisposableBase
{
	private IntPtr unmanagedMemory;
    private Stream stream;
    private bool isDisposed;
    
    protected override void Dispose(bool disposing)
    {      
        if (disposing)
        {
            // Dispose managed resources
            stream?.Dispose();
        }
        
        // Free unmanaged resources here
		if(unmanagedMemory != IntPtr.Zero)
		{
			Marshal.Free(unmanagedMemory);
		}
    }
}
```

### 2.2 ObservableBase<T>

Thread-safe IObservable implementation with subscription management:

```csharp
public abstract class ObservableBase<T> : DisposableBase, IObservable<T>
{
    // Thread-safe subscription list
    // Automatic cleanup on dispose
    
    protected void PushData(in T data);      // Notify all observers
    protected void PushError(Exception ex);  // Push error to observers
    protected void UnsubscribeAll();         // Clear all subscriptions
}
```

**Usage:**
```csharp
public class SensorDataStream : ObservableBase<SensorReading>
{
    public void OnNewReading(SensorReading reading)
    {
        PushData(reading);
    }
    
    protected override void Dispose(bool disposing)
    {
        UnsubscribeAll();
        base.Dispose(disposing);
    }
}
```

### 2.3 SimulationBase<TEntity, TId>

State machine for simulation lifecycle:

```csharp
public abstract class SimulationBase<TEntity, TId> : DisposableBase, ISimulation<TEntity, TId>
    where TEntity : class, ISimulationEntity<TId>
{
    public SimulationState State { get; }  // Thread-safe state access
    public Exception Error { get; }
    
    public void Initialize();  // Uninitialized → Initialized
    public void Start();       // Initialized → Running
    public void Pause();       // Running → Paused
    public void Resume();      // Paused → Running
    public void Stop();        // Running/Paused → Stopped
    
    // Override these for custom behavior
    protected abstract void OnInitialize();
    protected abstract void OnStart();
    protected abstract void OnPause();
    protected abstract void OnResume();
    protected abstract void OnStop();
}
```

### 2.4 LogWriterBase

Asynchronous log writer with queue-based processing:

```csharp
public abstract class LogWriterBase : ILogWriter
{
    public abstract string Id { get; }
    
    public void Start();
    public void Stop();
    public void WriteToLog(string message, LogLevel logLevel);
    
    // Override for custom output
    protected abstract void WriteLogEntry(in LogEntry entry);
    protected virtual void OnStart() { }
    protected virtual void OnStop() { }
}
```

### 2.5 SettingsBase

Base class for serializable settings:

```csharp
public abstract class SettingsBase : ISettings
{
    public abstract void Copy(ISettings source);
    public abstract void SetDefaults();
}
```

---

## 3. Dependency Injection

### 3.1 Attribute-Based Registration

Mark classes for automatic DI registration:

```csharp
// Register as interface implementation
[Export(typeof(IMyService))]
public class MyService : IMyService { }

// Register with specific lifetime
[Export(typeof(IMyService))]
[Singleton]
public class MySingletonService : IMyService { }

[Export(typeof(IMyService))]
[Scoped]
public class MyScopedService : IMyService { }

// Default lifetime is Transient (no attribute)

// Register with key for keyed services
[Export(typeof(IMyService), key: "primary")]
public class PrimaryService : IMyService { }

// Multiple exports from same class
[Export(typeof(IService1))]
[Export(typeof(IService2))]
public class MultiService : IService1, IService2 { }
```

### 3.2 Service Registration

```csharp
// Automatic service discovery from an assembly
var services = new ServiceCollection();
services.AddExportedServices(typeof(MyService).Assembly);

// Register from multiple assemblies
services.AddExportedServices(new[] { assembly1, assembly2 });

// Build standard provider
var provider = services.BuildServiceProvider();

// Or build dynamic provider (supports runtime additions)
var dynamicProvider = services.BuildDynamicServiceProvider();
```

### 3.3 Dynamic Service Provider

Supports adding services at runtime:

```csharp
var provider = new DynamicServiceProvider();
var serviceLoader = provider.GetRequiredService<IServiceLoader>();

// Add services dynamically
serviceLoader.AddExportedServices(typeof(MyService).Assembly);
serviceLoader.ConfigureEnvironmentInformation();

// Now resolve services
var service = provider.GetRequiredService<IMyService>();
```

### 3.4 Constructor Injection

```csharp
// Single constructor - auto-detected
public class MyService : IMyService
{
    public MyService(IDependency1 dep1, IDependency2 dep2) { }
}

// Multiple constructors - mark with attribute
public class MyService : IMyService
{
    public MyService() { }
    
    [ActivatorUtilitiesConstructor]
    public MyService(IDependency dep) { }
}

// Keyed service injection
public class MyService : IMyService
{
    public MyService([FromKeyedServices("primary")] IOtherService service) { }
}

// Optional service injection
public class MyService : IMyService
{
    public MyService([Optional] IOtherService service) { }
}

```

### 3.5 Configuration Extensions

```csharp
services.ConfigureEnvironmentInformation(builder =>
{
    builder.SetApplicationName("MyApp");
    builder.SetVersion("1.0.0");
});

services.ConfigureLoggerService(config =>
{
    config.MinimumLogLevel = LogLevel.Debug;
});
```

---

## 4. Result Pattern

### 4.1 Basic Usage

Use `Result` and `Result<T>` for operations that can fail:

```csharp
// Without return value
public Result ValidateUser(User user)
{
    if (user == null)
    {
        return Result.Failure("User cannot be null");
    }
    
    if (string.IsNullOrEmpty(user.Email))
    {
        return Result.Failure("Email is required", code: 1001);
    }
    
    return Result.Success();
}

// With return value
public Result<User> GetUser(int id)
{
    var user = repository.Find(id);
    if (user == null)
    {
        return Result<User>.Failure("User not found", statusCode: 404);
    }
    
    return Result<User>.Success(user);
}
```

### 4.2 Error Information

```csharp
public readonly struct ErrorInformation
{
    public readonly string Message;
    public readonly int Code;
    public readonly Exception Exception;
    public readonly string AdditionalDataJson;
}

// Create with exception
var error = new ErrorInformation(
    exception: ex,
    message: "Operation failed",
    code: 500
);

// Create with additional data
var error = new ErrorInformation(
    message: "Validation failed",
    code: 400,
    additionalDataJson: JsonSerializer.Serialize(validationErrors)
);
```

### 4.3 Error Propagation

```csharp
public Result<Order> ProcessOrder(OrderRequest request)
{
    // Validate request
    var validationResult = ValidateRequest(request);
    if (validationResult.IsFailure)
    {
        return Result<Order>.Failure(validationResult);  // Bubble up error
    }
    
    // Get user
    var userResult = GetUser(request.UserId);
    if (userResult.IsFailure)
    {
        return Result<Order>.Failure(userResult);  // Bubble up error
    }
    
    // Create order
    var order = CreateOrder(userResult.Value, request);
    return Result<Order>.Success(order);
}
```

### 4.4 Consuming Results

```csharp
var result = GetUser(userId);

if (result.IsSuccess)
{
    var user = result.Value;
    // Process user
}
else
{
    var error = result.Error.Value;
    logger.Error($"Failed: [{error.Code}] {error.Message}");
}
```

---

## 5. Threading Patterns

### 5.1 ReaderWriterLockSafe

Async-safe reader-writer lock returning handles:

```csharp
private readonly ReaderWriterLockSafe rwLock = new ReaderWriterLockSafe();

public Data ReadData()
{
    var handle = rwLock.EnterReadLock();
    try
    {
        return data;
    }
    finally
    {
        rwLock.ExitReadLock(handle);
    }
}

public void WriteData(Data newData)
{
    var handle = rwLock.EnterWriteLock();
    try
    {
        data = newData;
    }
    finally
    {
        rwLock.ExitWriteLock(handle);
    }
}

// Upgrade from read to write
public void ConditionalUpdate()
{
    var readHandle = rwLock.EnterReadLock();
    try
    {
        if (NeedsUpdate())
        {
            var writeHandle = rwLock.EnterWriteLock(readHandle);
            try
            {
                PerformUpdate();
            }
            finally
            {
                rwLock.ExitWriteLock(writeHandle);
            }
        }
    }
    finally
    {
        rwLock.ExitReadLock(readHandle);
    }
}
```

### 5.2 Dispatcher Pattern

Execute actions on main thread:

```csharp
public interface IDispatcher
{
    int Dispatch(Action action);
    int Dispatch<T>(Action<T> action, T argument);
}

// Usage
dispatcher.Dispatch(() => UpdateUI());
dispatcher.Dispatch(data => ProcessData(data), myData);
```

### 5.3 Job Scheduling

```csharp
public interface IJobScheduler
{
    JobHandle Schedule(IJob job);
    void Cancel(JobHandle handle);
}

public interface IJob
{
    void Execute();
}
```

---

## 6. Collections

### 6.1 Object Pooling

Reduce allocations with object pools:

```csharp
// Create internal/locally used pools via factory (IPoolFactory)
IPool<MyObject> pool = poolFactory.Create<MyObject>(
    synchronized: true,
    allocator: () => new MyObject(),
    releaser: obj => obj.Reset(),
    destructor: obj => obj.Dispose()
);

// Create shared/publicly used pools via service (IPoolService).
IPool<MyObject> pool = poolService.Create<MyObject>(
    synchronized: true,
    allocator: () => new MyObject(),
    releaser: obj => obj.Reset(),
    destructor: obj => obj.Dispose()
);

// Use pooled objects
var obj = pool.Allocate();
try
{
    obj.DoWork();
}
finally
{
    pool.Release(obj);
}

// Monitor pool health
Console.WriteLine($"Available: {pool.Available}/{pool.Capacity}");
Console.WriteLine($"Drop rate: {pool.DropRate:P}");  // Objects destroyed when pool full
```

### 6.2 Data Caching

```csharp
IDataCache<string, User> cache = cacheFactory.Create<string, User>(
    synchronized: true,
    capacity: 100
);

cache.Add("user-123", user);
var cached = cache["user-123"];

if (cache.Contains("user-123"))
{
    cache.Remove("user-123");
}

cache.Clear();
```

### 6.3 Circular List

Fixed-capacity list with automatic overwrite of oldest items:

```csharp
var history = new CircularList<LogEntry>(capacity: 100);

// Add items (oldest automatically removed when full)
history.Push(entry1);
history.Push(entry2);

// Access by index (0 = oldest)
var oldest = history[0];
var newest = history[history.Count - 1];

// Pop from front
var removed = history.Pop();

// Iterate
foreach (var entry in history)
{
    Process(entry);
}
```

---

## 7. Value Types and Structs

### 7.1 Readonly Structs

Use for immutable value types:

```csharp
public readonly struct Range<T> : IEquatable<Range<T>>
    where T : struct, IComparable<T>, IEquatable<T>
{
    public readonly T Minimum;
    public readonly T Maximum;
    
    public Range(in T a, in T b)
    {
        // Auto-sort on construction
        if (a.CompareTo(b) <= 0)
        {
            Minimum = a;
            Maximum = b;
        }
        else
        {
            Minimum = b;
            Maximum = a;
        }
    }
    
    public bool Contains(T value) => 
        Minimum.CompareTo(value) <= 0 && Maximum.CompareTo(value) >= 0;
    
    public T Clamp(T value) { ... }
    public Range<T> Intersect(in Range<T> other) { ... }
    public Range<T> Union(in Range<T> other) { ... }
}
```

### 7.2 Using `in` Parameters

Pass structs by reference to avoid copying:

```csharp
// Good - no copy
public void Process(in Range<double> range) { }
public static Result<T> Failure(in ErrorInformation error) { }

// Also for returning values via ref
protected void PushData(in T data) { }
```

---

## 8. Factory Pattern

### 8.1 Interface-Based Factories

```csharp
public interface IPoolFactory
{
    IPool<T> Create<T>(
        bool synchronized,
        Func<T> allocator,
        Action<T> releaser = null,
        Action<T> destructor = null)
        where T : class;
}

public interface IDataCacheFactory
{
    IDataCache<TKey, TValue> Create<TKey, TValue>(
        bool synchronized,
        int capacity)
        where TKey : IEquatable<TKey>;
}

public interface IBinaryReaderWriterFactory
{
    IBinaryReader CreateReader(Stream stream);
    IBinaryWriter CreateWriter(Stream stream);
}
```

### 8.2 Factory Registration

```csharp
[Export(typeof(IPoolFactory))]
[Singleton]
internal sealed class PoolFactory : IPoolFactory
{
    public IPool<T> Create<T>(...) where T : class
    {
        return synchronized 
            ? new SynchronizedPool<T>(allocator, releaser, destructor)
            : new Pool<T>(allocator, releaser, destructor);
    }
}
```

---

## 9. Serialization

### 9.1 ISerializableObject

Standard interface for serializable objects:

```csharp
public interface ISerializableObject
{
    void Serialize(string filePath);
    void Serialize(Stream stream);
    void Deserialize(string filePath);
    void Deserialize(Stream stream);
}
```

### 9.2 Binary Serialization

```csharp
public interface IBinaryWriter
{
    void Write(bool value);
    void Write(int value);
    void Write(float value);
    void Write(string value);
    // ... more types
}

public interface IBinaryReader
{
    bool ReadBoolean();
    int ReadInt32();
    float ReadSingle();
    string ReadString();
    // ... more types
}
```

---

## 10. Testing Patterns

### 10.1 Test Structure

Follow arrange-act-assert with comments:

```csharp
[TestFixture]
internal class PoolTests
{
    [Test]
    public void Pool_Allocate_ReturnsNewInstance()
    {
        // arrange
        var sut = new Pool<TestObject>(
            () => new TestObject(),
            releaser: null,
            destructor: null);
        
        // act
        var result = sut.Allocate();
        
        // assert
        Assert.That(result, Is.Not.Null);
        Assert.That(sut.Available, Is.EqualTo(0));
    }
}
```

### 10.2 Test Naming

`Method_Scenario_ExpectedResult` or `Method_Condition_Behavior`:

```csharp
[Test]
public void Pool_Constructor_Ok() { }

[Test]
public void Pool_Constructor_No_Allocator() { }

[Test]
public void Pool_Allocate_Release_Allocate_with_onRelease() { }
```

### 10.3 Mocking with NSubstitute

```csharp
[Test]
public void Service_WithDependency_CallsDependency()
{
    // arrange
    var mockDependency = Substitute.For<IDependency>();
    mockDependency.GetValue().Returns(42);
    
    var sut = new MyService(mockDependency);
    
    // act
    sut.Process();
    
    // assert
    mockDependency.Received(1).GetValue();
}
```

### 10.4 Testing DI Registration

```csharp
[Test]
public void Build_Core_ServiceProvider()
{
    // arrange
    var sut = new ServiceCollection();
    
    // act
    sut.AddExportedServices(typeof(Check).Assembly);
    var result = sut.BuildServiceProvider();
    
    // assert
    Assert.That(result, Is.Not.Null);
}

[Test]
public void DynamicServiceProvider_LoggerService_Build()
{
    // arrange
    var services = new ServiceCollection();
    services.AddExportedServices(typeof(Check).Assembly);
    services.ConfigureEnvironmentInformation();
    var sut = services.BuildDynamicServiceProvider();
    
    // act
    var result = sut.GetRequiredService<ILoggerService>();
    
    // assert
    Assert.That(result, Is.Not.Null);
}
```

---

## 11. Interface Design

### 11.1 Internal Interface Methods

Use C# 8.0+ default interface methods for framework internals:

```csharp
public interface IDispatcher
{
    // Public API
    int Dispatch(Action action);
    int Dispatch<T>(Action<T> action, T argument);
    
    // Internal methods - visible only within assembly
    internal void Start();
    internal void Stop();
    internal void ExecuteDispatchedTasks(int executionTimeoutMs);
}
```

### 11.2 Interface Hierarchies

```csharp
public interface IDataCacheBase<TKey>
{
    int Count { get; }
    int Capacity { get; }
    bool Contains(TKey key);
    void Remove(TKey key);
    void Clear();
}

public interface IDataCache<TKey, TValue> : IDataCacheBase<TKey>
{
    TValue this[TKey key] { get; set; }
    void Add(TKey key, TValue value);
}
```

---

## Quick Reference

### Pattern Selection Guide

| Scenario | Pattern |
|----------|---------|
| Resource cleanup needed | `DisposableBase` |
| Event streaming | `ObservableBase<T>` |
| UI separation | `PresenterBase<TView>` |
| State machine lifecycle | `SimulationBase<TEntity, TId>` |
| Operation can fail | `Result` / `Result<T>` |
| Need object reuse | `IPool<T>` via factory |
| Async-safe locking | `ReaderWriterLockSafe` |
| Service registration | `[Export]` + lifetime attributes |

### Validation Quick Reference

```csharp
// Public API boundary
Check.ThrowIfNull(param, nameof(param));

// Debug-only internal check  
Assertion.ThrowIfNull(internalState, nameof(internalState));

// Never throw exceptions for expected failures - use Result pattern
return Result.Failure("Expected error");
```

---
