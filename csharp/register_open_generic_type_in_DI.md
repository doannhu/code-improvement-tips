# Registering Open Generic Types in .NET Dependency Injection

## Overview
In .NET's `IServiceCollection`, you can register **open generic types** so the DI container can resolve any closed generic instance automatically at runtime.

## Code Sample

```csharp
// Interface
public interface IUploadCentreEventLogger<T>
{
    Guid? ProcessId { get; set; }
    string Filename { get; set; }
    string Source { get; set; }
}

// Implementation
public class UploadCentreEventLogger<T> : IUploadCentreEventLogger<T>
{
    public Guid? ProcessId { get; set; } = Guid.NewGuid();
    public string Filename { get; set; }
    public string Source { get; set; }

    public UploadCentreEventLogger()
    {
        Source = typeof(T).Name;
    }
}

// Registration in IServiceCollection
services.AddTransient(typeof(IUploadCentreEventLogger<>), typeof(UploadCentreEventLogger<>));

// Usage
public class MyService
{
    private readonly IUploadCentreEventLogger<MyService> _logger;

    public MyService(IUploadCentreEventLogger<MyService> logger)
    {
        _logger = logger;
    }
}
```

# Recommendation
- Prefer generic interfaces (e.g., IUploadCentreEventLogger<T>) over non-generic ones for better type safety.
- Register open generics with typeof(IInterface<>) and typeof(Implementation<>).
- Avoid manual new for generic services — let DI resolve them automatically.
- This pattern works well for loggers, repositories, and type-specific services.
