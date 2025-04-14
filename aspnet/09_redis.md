# Лабораторная работа: C# ASP.NET Core API с Redis и Entity Framework Core

## Цель работы
Научиться создавать веб-API на ASP.NET Core с использованием Redis для кэширования и Entity Framework Core для работы с базой данных.

## Теоретическая часть

### Redis
Redis - это хранилище данных типа "ключ-значение" в памяти, которое можно использовать в качестве базы данных, кэша или брокера сообщений. В этом проекте мы будем использовать Redis для кэширования данных, чтобы уменьшить нагрузку на основную базу данных.

### Entity Framework Core
Entity Framework Core - это ORM (Object-Relational Mapper) для .NET, который позволяет работать с базой данных, используя объекты .NET.

## Практическая часть

### 1. Создание проекта

1. Создайте новый проект ASP.NET Core Web API:

### 2. Установка необходимых пакетов

Добавьте следующие NuGet пакеты:
- Microsoft.EntityFrameworkCore.SqlServer
- Microsoft.EntityFrameworkCore.Design
- StackExchange.Redis
- Microsoft.Extensions.Caching.StackExchangeRedis

### 3. Настройка Redis

В `Program.cs` добавьте сервис Redis:
```csharp
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "RedisLab_";
});
```

### 4. Настройка базы данных

Добавьте в `appsettings.json` строки подключения:
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=RedisLabDb;Trusted_Connection=True;",
    "Redis": "localhost:6379"
  }
}
```

### 5. Создание модели и контекста базы данных

Создайте папку `Models` и добавьте класс `Product`:
```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public int Stock { get; set; }
}
```

Создайте класс `AppDbContext`:
```csharp
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options)
    {
    }

    public DbSet<Product> Products { get; set; }
}
```

Добавьте контекст в `Program.cs`:
```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
```

### 6. Создание репозитория

Создайте папку `Repositories` и добавьте интерфейс и реализацию:
```csharp
public interface IProductRepository
{
    Task<Product> GetByIdAsync(int id);
    Task<List<Product>> GetAllAsync();
    Task AddAsync(Product product);
    Task UpdateAsync(Product product);
    Task DeleteAsync(int id);
}

public class ProductRepository : IProductRepository
{
    private readonly AppDbContext _context;
    private readonly IDistributedCache _cache;
    private readonly ILogger<ProductRepository> _logger;

    public ProductRepository(AppDbContext context, IDistributedCache cache, ILogger<ProductRepository> logger)
    {
        _context = context;
        _cache = cache;
        _logger = logger;
    }

    public async Task<Product> GetByIdAsync(int id)
    {
        string cacheKey = $"product_{id}";
        
        try
        {
            // Попытка получить данные из кэша
            string cachedProduct = await _cache.GetStringAsync(cacheKey);
            if (cachedProduct != null)
            {
                _logger.LogInformation($"Product {id} found in cache");
                return JsonSerializer.Deserialize<Product>(cachedProduct);
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Redis error occurred. Falling back to database.");
        }

        // Если нет в кэше, получаем из базы данных
        var product = await _context.Products.FindAsync(id);
        if (product == null) return null;

        // Сохраняем в кэш
        try
        {
            var cacheOptions = new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
            };
            
            await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(product), cacheOptions);
            _logger.LogInformation($"Product {id} added to cache");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to set product to Redis cache");
        }

        return product;
    }

    // Остальные методы реализации...
}
```

Добавьте ProductRepository в `Program.cs`:
```csharp
builder.Services.AddScoped<IProductRepository, ProductRepository>();
```

### 7. Создание контроллера

Создайте контроллер `ProductsController`:
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductRepository _repository;
    private readonly ILogger<ProductsController> _logger;

    public ProductsController(IProductRepository repository, ILogger<ProductsController> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<Product>> GetById(int id)
    {
        _logger.LogInformation($"Getting product {id}");
        var product = await _repository.GetByIdAsync(id);
        if (product == null) return NotFound();
        return product;
    }

    // Другие действия...
}
```

### 8. Миграции базы данных

Выполните команды для создания и применения миграций:
```bash
dotnet ef migrations add InitialCreate
dotnet ef database update
```

## Задания для самостоятельной работы

1. Реализуйте остальные методы в `ProductRepository` (GetAllAsync, AddAsync, UpdateAsync, DeleteAsync) с поддержкой кэширования.
2. Выгрузите проект на GitHub.

## Дополнительное задание
1. Создайте сервис, который будет обновлять кэш в фоновом режиме.

