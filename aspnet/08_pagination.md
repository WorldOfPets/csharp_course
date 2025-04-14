# Лабораторная работа: Реализация пагинации в ASP.NET Core API с Entity Framework Core

## 1. Создание и настройка проекта

### 1.1. Создание проекта

### 1.2. Добавление необходимых NuGet пакетов
- Microsoft.EntityFrameworkCore
- Microsoft.EntityFrameworkCore.SqlServer
- Microsoft.EntityFrameworkCore.Design
- Microsoft.EntityFrameworkCore.Tools

### 1.3. Настройка Program.cs
```csharp
var builder = WebApplication.CreateBuilder(args);

// Добавление сервисов в контейнер
builder.Services.AddControllers();

// Настройка Entity Framework Core
builder.Services.AddDbContext<ApplicationDbContext>(options => {
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection"));
    options.ConfigureWarnings(warnigs => warnigs.Ignore(RelationalEventId.PendingModelChangesWarning));
    });

var app = builder.Build();

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

## 2. Настройка базы данных

### 2.1. Добавление строки подключения в appsettings.json
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=PaginationLabDB;Trusted_Connection=True;"
  },
  "AllowedHosts": "*"
}
```

### 2.2. Создание класса контекста базы данных
```csharp
using Microsoft.EntityFrameworkCore;

public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) 
        : base(options)
    {
    }

    public DbSet<Product> Products { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        var products = new List<Product>();
        var baseDate = new DateTime(2025, 1, 1); // Static base date
        
        for (int i = 1; i <= 100; i++)
        {
            products.Add(new Product
            {
                Id = i,
                Name = $"Product {i}",
                Price = i * 10,
                CreatedDate = baseDate.AddDays(-i) // Now using static base
            });
        }
        
        modelBuilder.Entity<Product>().HasData(products);
    }
}
```

## 3. Модель данных

### 3.1. Модель Product
```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public DateTime CreatedDate { get; set; }
}
```

## 4. Классы для пагинации

### 4.1. Параметры пагинации и фильтрации
```csharp
public class ProductParameters
{
    const int maxPageSize = 50;
    public int PageNumber { get; set; } = 1;
    private int _pageSize = 10;
    
    public int PageSize
    {
        get => _pageSize;
        set => _pageSize = (value > maxPageSize) ? maxPageSize : value;
    }
    
    // Параметры для сортировки
    public string OrderBy { get; set; } = "name"; // По умолчанию сортировка по имени
    
    // Параметры для фильтрации
    public decimal? MinPrice { get; set; }
    public decimal? MaxPrice { get; set; }
}
```

### 4.2. Класс PagedList
```csharp
public class PagedList<T> : List<T>
{
    public int CurrentPage { get; private set; }
    public int TotalPages { get; private set; }
    public int PageSize { get; private set; }
    public int TotalCount { get; private set; }
    
    public bool HasPrevious => CurrentPage > 1;
    public bool HasNext => CurrentPage < TotalPages;
    
    public PagedList(List<T> items, int count, int pageNumber, int pageSize)
    {
        TotalCount = count;
        PageSize = pageSize;
        CurrentPage = pageNumber;
        TotalPages = (int)Math.Ceiling(count / (double)pageSize);
        AddRange(items);
    }
    
    public static async Task<PagedList<T>> ToPagedListAsync(IQueryable<T> source, 
        int pageNumber, int pageSize)
    {
        var count = await source.CountAsync();
        var items = await source.Skip((pageNumber - 1) * pageSize)
                               .Take(pageSize)
                               .ToListAsync();
        return new PagedList<T>(items, count, pageNumber, pageSize);
    }
}
```

## 5. Контроллер Products

### 5.1. ProductsController
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly ApplicationDbContext _context;
    
    public ProductsController(ApplicationDbContext context)
    {
        _context = context;
    }
    
    [HttpGet]
    public async Task<ActionResult<PagedList<Product>>> GetProducts(
        [FromQuery] ProductParameters productParameters)
    {
        var products = _context.Products.AsQueryable();
        
        var pagedProducts = await PagedList<Product>.ToPagedListAsync(
            products,
            productParameters.PageNumber,
            productParameters.PageSize);
        
        // Метаданные пагинации
        var metadata = new
        {
            pagedProducts.TotalCount,
            pagedProducts.PageSize,
            pagedProducts.CurrentPage,
            pagedProducts.TotalPages,
            pagedProducts.HasNext,
            pagedProducts.HasPrevious
        };
        
        Response.Headers.Add("X-Pagination", JsonSerializer.Serialize(metadata));
        
        return Ok(pagedProducts);
    }
}
```

## 6. Применение миграций и создание базы данных

### 6.1. Создание миграции
```bash
dotnet ef migrations add InitialCreate
```

### 6.2. Применение миграции
```bash
dotnet ef database update
```

## 7. Примеры запросов

### 7.1. Получение первой страницы (10 элементов)
```
GET /api/products
```

### 7.2. Получение второй страницы с размером 5 элементов
```
GET /api/products?pageNumber=2&pageSize=5
```
# Задания
1. Перенесите метаданные пагинации в тело запроса.
2. Реализуйте сортировку и фильтрацию данных.
3. Вынесите создание тестовых данных в другое место (seed controller or db initializer)
4. Загрузите проект на GitHub.
5. **Интегрируйте пагинацию в свой StoreApi**

## Доп. задание
1. Реализуйте пагинацию через библиотеку.
