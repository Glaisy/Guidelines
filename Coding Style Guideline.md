# Space Development Coding Style Guideline

## Overview

This document defines the **formatting, layout, and structural conventions** for the Space Development projects.
Focus is on code organization, spacing, indentation, naming conventions, member ordering, and StyleCop rules.

---

## 1. File Structure & Layout

### 1.1 Using Directives

```csharp
using System.Reflection;

using Microsoft.Extensions.DependencyInjection;

using Space.Core.DependencyInjection.Internals;
using Space.Core.Diagnostics;

namespace Space.Core.DependencyInjection;
```

**Rules:**
- Using directives **outside namespace** (not inside)
- One blank line before namespace declaration
- Group and order:
  1. System namespaces
  2. Project and Third-party namespaces should be ordered alphabetically and separated with one blank line

- Alphabetically sorted within each group
- Common System usings defined in `Directory.Build.props` (don't repeat)

### 1.2 Namespace Declaration

**File-scoped namespaces only** (C# 10+):

```csharp
namespace Space.Core.Collections;

public class CircularList<T>
{
}
```

**NOT:**
```csharp
// ❌ DO NOT USE
namespace Space.Core.Collections
{
    public class CircularList<T>
    {
    }
}
```

### 1.3 One Type Per File

Each file must contain **exactly one top-level type** (class, struct, enum, interface, delegate, or record). Do **not** define multiple types in a single file.

**Exceptions:**
- **Nested types** are allowed inside their enclosing type
- **Partial classes** split across files (see [Section 9.1](#91-partial-classes))

```csharp
// ✅ CORRECT - One type per file
// CircularList.cs
namespace Space.Core.Collections;

public class CircularList<T>
{
    // Nested type is fine
    public struct Enumerator : IEnumerator<T>
    {
    }
}
```

```csharp
// ❌ INCORRECT - Multiple top-level types in one file
// Collections.cs
namespace Space.Core.Collections;

public class CircularList<T>
{
}

public class Stack<T>  // ❌ Should be in its own file
{
}

public enum SortOrder  // ❌ Should be in its own file
{
    Ascending,
    Descending,
}
```

---

## 2. Indentation & Spacing

### 2.1 Basic Rules

- **Indent Size**: 4 spaces
- **Use spaces only** (no tabs)
- **Line Endings**: LF (Unix-style)
- **Encoding**: UTF-8

### 2.2 Blank Lines

#### Between Class Sections

Use **TWO blank lines** between major class sections:

```csharp
public class Example
{
    private const int Constant = 10;
    
    
    private readonly int field;
    private int anotherField;


    public Example()
    {
    }


    public int Property { get; set; }


    public void Method1()
    {
    }

    public void Method2()
    {
    }


    private void PrivateMethod()
    {
    }
}
```

**Key Points:**
- **Two blank lines** between: fields→constructors, constructors→properties, properties→methods, 
	public properties/methods→internal properties/methods→protected properties/methods→private properties/methods
- **One blank line** between methods within same access level
- **One blank line** between field groups (constants, readonly, regular)
- **No blank line** at start/end of class body, between property backing field and XML command/property implementation

#### Within Methods

```csharp
public void Process()
{
    var item = GetItem();
    
    if (item == null)
    {
        return;
    }

    // process item
    item.Validate();
    item.Execute();
}
```

Use blank lines **sparingly** to separate logical blocks—only when it improves readability.

### 2.3 Horizontal Spacing

```csharp
// Binary operators - spaces around
x = a + b;
if (value == 0)

// Method calls - no space before parenthesis
MyMethod(a, b, c);

// Parameters - space after comma
public void Method(int a, int b, int c)

// Type constraints - space before/after
where T : class

// Generic types - no space
List<int>
Dictionary<string, int>
```

---

## 3. Naming Conventions

### 3.1 Casing Rules

| Element | Convention | Example |
|---------|-----------|---------|
| **Namespace** | PascalCase | `Space.Core.Collections` |
| **Class** | PascalCase | `CircularList`, `PoolBase` |
| **Interface** | IPascalCase | `IDispatcher`, `IPool` |
| **Struct** | PascalCase | `Result`, `ErrorInformation` |
| **Enum** | PascalCase | `LogLevel`, `OperatingSystemPlatform` |
| **Enum Member** | PascalCase | `LogLevel.Error` |
| **Method** | PascalCase | `GetEnumerator`, `AllocateItem` |
| **Property** | PascalCase | `Capacity`, `IsSuccess` |
| **Event** | PascalCase | `PropertyChanged` |
| **Delegate** | PascalCase | `Action`, `EventHandler` |
| **Constant (const)** | PascalCase | `InitialCapacity` |
| **Private Static readonly field**  | camelCase | `maxValue` or `defaultValue` |
| **Private Static readonly field** as constants | PascalCase | `MaxValue` or `DefaultValue` |
| **Public/Protected Static readonly field** | PascalCase | `MaxValue` or `DefaultValue` |
| **Private field** | camelCase | `buffer`, `version`, `head` |
| **Parameter** | camelCase | `capacity`, `value`, `name` |
| **Local variable** | camelCase | `index`, `result`, `item` |
| **Generic type parameter** | T, TKey, TValue | `CircularList<T>`, `IPool<T>` |

### 3.2 Special Patterns

```csharp
// Base classes - suffix with "Base"
public abstract class DisposableBase
public abstract class PoolBase<T>
public abstract class CanvasBase

// Internal implementations - in "Internals" folder
namespace Space.Core.Collections.Internals;
internal sealed class Pool<T> : PoolBase<T>

// Test classes - suffix with "Tests"
public class UnitConverterTests
public class PoolFactoryTests

// Extension classes - suffix with "Extensions"
public static class ServiceCollectionExtensions
public static class StringExtensions

// Attribute classes - suffix with "Attribute"
public sealed class ExportAttribute : Attribute

// Interfaces - prefix with "I"
public interface IDispatcher
public interface IPool<T>
```

### 3.3 Field Naming

```csharp
public class Example
{
    // Constants - PascalCase
    private const int DefaultCapacity = 16;
    private const float DropRateLimit = 0.3f;
    
    
    // Readonly fields - camelCase
    private readonly Func<T> allocator;
    private readonly Action<T> releaser;
    private readonly List<T> items;
    
    // Regular fields - camelCase
    private int releaseCount;
    private int dropCount;
    private bool isDisposed;
}
```

**No prefixes** like `_`, `m_`, `s_` for fields.

### 3.4 Type Aliases vs BCL Types

Use the C# keyword aliases for type declarations and the BCL type names (PascalCase) for static method calls:
Always the framework type for IntPtr and UIntPtr64 declarations.

```csharp
// ✅ CORRECT - Use keyword alias for declarations
string name;
int count;
bool isValid;
IntPtr pointer;

// ✅ CORRECT - Use BCL type name for static method calls
var pointer = IntPtr.Zero;
if (String.IsNullOrEmpty(name)) { }
var text = String.Join(", ", items);
var number = Int32.Parse(value);

// ❌ INCORRECT - Do not use keyword alias for static methods
nint pointer;
if (string.IsNullOrEmpty(name)) { }
var text = string.Join(", ", items);
```

This convention makes it clear when you're calling a static method on the type vs using the type for a declaration.

### 3.5 Implicit Typing (`var`)

Use the `var` keyword for all local variable declarations instead of explicit type names:

```csharp
// ✅ CORRECT - Use var
var items = new List<string>();
var count = GetCount();
var name = "Hello";
var result = Process(input);
var stream = File.OpenRead(path);
var pair = new KeyValuePair<string, int>("key", 1);

// ❌ INCORRECT - Do not use explicit type
List<string> items = new List<string>();
int count = GetCount();
string name = "Hello";
Result result = Process(input);
FileStream stream = File.OpenRead(path);
KeyValuePair<string, int> pair = new KeyValuePair<string, int>("key", 1);
```

This applies to all local variable declarations where the type can be inferred by the compiler. The `var` keyword keeps code concise and reduces visual redundancy.

**Note:** This rule applies to **local variables only**. Field declarations, parameter types, return types, and other member declarations must still use explicit types.

```csharp
public class Example
{
    // Fields still use explicit types
    private readonly List<string> items;


    public Example()
    {
        // Local variables use var
        var list = new List<string>();
        items = list;
    }


    // Return types and parameters still use explicit types
    public List<string> GetItems(int count)
    {
        var result = new List<string>(count);
        return result;
    }
}
```

---

## 4. Class Member Ordering

### 4.1 Standard Order

```csharp
public class StandardClass
{
     // 1. Private/Protected/Public Nested types (enums, classes, structs)
    public enum NestedEnum
    {
    }

	// 2. Constants (Public first, then protected, then private)
    public const int PublicConstant = 1;
	protected const int ProtectedConstant = 2;
    private const int PrivateConstant = 3;
    
    
    // 3. Static readonly fields
    private static readonly object syncLock;
    
    
    // 4. Instance readonly fields (private/internal/protected/public)
    private readonly int readonlyField;
    
    
    // 5. Instance fields (private/internal/protected/public)
    private int field;
    private bool isActive;


    // 6. Constructors, Finalizer, Dispose (static first, then instance)
    public StandardClass()
    {
    }

	~StandardClass()
	{
		Dispose(false);
	}
	
	public void Dispose()
	{
		Dispose(true);
		GC.SuppressFinalize(this);
	}
	
	protected overrider Dispose(bool disposing)
	{
	}
	

    // 7. Public Properties
    public int PublicProperty { get; set; }


    // 8. Public methods
    public void PublicMethod()
    {
    }


    // 9. Internal Properties
    internal int InternalProperty { get; }


    // 10. Internal methods
    internal void InternalMethod()
    {
    }


    // 11. Protected Properties
    protected int ProtectedProperty { get; set; }

    // 12. Protected methods
    protected virtual void ProtectedMethod()
    {
    }


    // 13. Private Properties
    private int PrivateProperty { get; set; }


    // 14. Private methods
    private void PrivateMethod()
    {
    }
}
```

IMPORTANT: Properties and methods inside a section should be ordered each other in ascending alphabetical order.

### 4.2 Flexibility

Logical grouping is allowed:
- Related methods can be grouped together
- Properties can be near their backing fields
- Nested types can be with related members

---

## 5. Braces & Control Structures

### 5.1 Brace Placement

**Always on same line** (K&R style):

```csharp
public class MyClass
{
    public void Method()
    {
        if (condition)
        {
            DoSomething();
        }
        else
        {
            DoSomethingElse();
        }
    }
}
```

### 5.2 Always Use Braces

**Required** even for single statements:

```csharp
// ✅ CORRECT
if (value < 0)
{
    throw new ArgumentException();
}

// ❌ INCORRECT
if (value < 0)
    throw new ArgumentException();
```

### 5.3 Control Structure Spacing

```csharp
// Keywords - space before opening brace
if (condition) { }
for (int i = 0; i < 10; i++) { }
while (running) { }
switch (value) { }

// No space before parenthesis
if(condition)  // ❌ WRONG
if (condition) // ✅ CORRECT
```

---

## 6. Expression-Bodied Members

### 6.1 Properties - Use Expression Body

```csharp
// ✅ DO
public int Capacity => buffer.Length;
public bool IsSuccess => !IsFailure;
public int Available => items.Count;

// Auto-properties
public int Count { get; private set; }
public bool Synchronized { get; }
```

### 6.2 Indexers - Use Expression Body

```csharp
// ✅ DO
public T this[int index] => buffer[index];

// More complex - use block body
public T this[int index]
{
    get
    {
        Check.ThrowIfLessThan(index, 0, nameof(index));
        return buffer[index];
    }
}
```

### 6.3 Methods - Do NOT Use Expression Body

```csharp
// ✅ CORRECT
public int GetValue()
{
    return value;
}

public static Result Success()
{
    return default;
}

// ❌ INCORRECT
public int GetValue() => value;
public static Result Success() => default;
```

### 6.4 Constructors - Do NOT Use Expression Body

```csharp
// ✅ CORRECT
public MyClass(int value)
{
    this.value = value;
}

// ❌ INCORRECT
public MyClass(int value) => this.value = value;
```

---

## 7. Special Cases

### 7.1 Partial Classes

Use for logical separation:

```csharp
// CircularList.cs
public partial class CircularList<T> : IReadOnlyList<T>
{
    // Main implementation
}

// CircularList.Enumerator.cs
public partial class CircularList<T>
{
    public struct Enumerator : IEnumerator<T>
    {
        // Enumerator implementation
    }
}
```

### 7.2 Public Interfaces with Internals Methods

```csharp
public interface IDispatcher
{
    // Public API
    int Dispatch(Action action);
    
    
    // Internal methods for framework use
    /// <summary>
    /// Start the dispatcher instance.
    /// </summary>
    internal void Start();
    
    /// <summary>
    /// Stops the dispatcher instance.
    /// </summary>
    internal void Stop();
}
```

### 7.3 Readonly Structs

```csharp
public readonly struct Result : IResult
{
    private Result(in ErrorInformation error)
    {
        Error = error;
        IsFailure = true;
    }


    public ErrorInformation? Error { get; }
    
    public bool IsFailure { get; }
    
    public bool IsSuccess => !IsFailure;


    public static Result Success()
    {
        return default;
    }
}
```

### 7.4 Generic Constraints

```csharp
// On same line if short
public class Pool<T> : IPool<T>
    where T : class
{
}

// Multiple constraints
public class Repository<TKey, TValue>
    where TKey : IComparable<TKey>
    where TValue : class, new()
{
}
```

---

## 8. Quick Reference

### ✅ DO

```csharp
// File-scoped namespaces
namespace Space.Core.Collections;

public class CircularList<T>
{
    // Two blank lines between sections
    private readonly T[] buffer;
    private int count;


    public CircularList(int capacity)
    {
        // Always use braces
        if (capacity < 2)
        {
            throw new ArgumentException();
        }

        // Use var for local variables
        var size = capacity * 2;
        buffer = new T[size];
    }


    // Expression-bodied properties
    public int Capacity => buffer.Length;
    
    // PascalCase for properties
    public int Count => count;


    // Methods with block body
    public void Clear()
    {
        count = 0;
    }
}
```

### ❌ DON'T

```csharp
// Traditional namespace
namespace Space.Core.Collections {  // ❌

public class circular_list<T> {  // ❌ camelCase class name
    private T[] _buffer;  // ❌ underscore prefix
    private int Count;  // ❌ PascalCase field

    public circular_list(int capacity)  // ❌ camelCase constructor
    {
        if (capacity < 2)  // ✅ Good
            throw new ArgumentException();  // ❌ No braces
        buffer = new T[capacity];
    }

    // ❌ Explicit type on local variable
    List<int> numbers = new List<int>();

    // ❌ Expression body on method
    public void Clear() => count = 0;

    }
}  // ❌ Closing brace for namespace
```

---

## Summary Checklist

Before committing code:

- [ ] **One type per file** (no multiple top-level types, except nested classes)
- [ ] **File-scoped namespace** used
- [ ] **Using directives outside** namespace, properly grouped
- [ ] **Two blank lines** between class sections
- [ ] **PascalCase** for types, methods, properties
- [ ] **camelCase** for fields, parameters, locals
- [ ] **`var` keyword** used for all local variable declarations
- [ ] **Braces always used** for control structures
- [ ] **Expression bodies** only for properties/indexers/accessors
- [ ] **BCL type names** for static method calls (`String.IsNullOrEmpty`, not `string.IsNullOrEmpty`)
- [ ] **4-space indentation** (spaces, not tabs)
- [ ] **No trailing whitespace**
- [ ] Code builds

---

**Last Updated**: 2026-06-23 
**Project**: Space.Core  
**Company**: Space Development
