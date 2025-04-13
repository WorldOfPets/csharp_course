# Лабораторная работа: Фильтрация и сортировка в ASP.NET Core API с Entity Framework Core

## Теоретическая часть

### Фильтрация данных
Фильтрация в API позволяет клиентам получать только те данные, которые соответствуют определенным критериям через query parameters.

### Сортировка данных
Сортировка позволяет клиентам указывать порядок возвращаемых данных по одному или нескольким полям.

## Практическая часть

### Задание 0: Создание проекта и настройка EF Core

1. Создайте новый проект

2. Добавьте необходимые пакеты:
- Microsoft.AspNetCore.Identity.EntityFrameworkCore
- Microsoft.EntityFrameworkCore.SqlServer
- Microsoft.EntityFrameworkCore.Tools
- System.Linq.Dynamic.Core

### 1. Создание моделей данных

```csharp
// Models/Category.cs
using System.ComponentModel.DataAnnotations;

namespace FilteringSortingApi.Models
{
    public class Category
    {
        [Key]
        public int Id { get; set; }
        
        [Required]
        [StringLength(50)]
        public string Name { get; set; }
        
        [StringLength(200)]
        public string Description { get; set; }
        public ICollection<Product> Products { get; set; }
    }
}

// Models/Product.cs
using System;
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace FilteringSortingApi.Models
{
    public class Product
    {
        [Key]
        public int Id { get; set; }
        
        [Required]
        [StringLength(100)]
        public string Name { get; set; }
        
        [ForeignKey("Category")]
        public int CategoryId { get; set; }
        
        public Category Category { get; set; }
        
        [Range(0, 10000)]
        public decimal Price { get; set; }
        
        public DateTime CreatedDate { get; set; } = DateTime.UtcNow;
        
        public bool InStock { get; set; } = true;
    }
}
```

### 2. Контекст базы данных

```csharp
// Data/AppDbContext.cs
using FilteringSortingApi.Models;
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;

namespace FilteringSortingApi.Data
{
    public class AppDbContext : IdentityDbContext
    {
        public AppDbContext(DbContextOptions<AppDbContext> options)
            : base(options)
        {
        }

        public DbSet<Product> Products { get; set; }
        public DbSet<Category> Categories { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            base.OnModelCreating(modelBuilder);
            
            modelBuilder.Entity<Product>()
                .Property(p => p.Price)
                .HasColumnType("decimal(18,2)");
                
            modelBuilder.Entity<Category>()
                .HasMany(c => c.Products)
                .WithOne(p => p.Category)
                .HasForeignKey(p => p.CategoryId);
        }
    }
}
```
1. Добавьте подключение к БД в `Program.cs`:
```csharp
// Добавьте перед builder.Build()
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));

// Добавьте Identity (опционально)
builder.Services.AddDefaultIdentity<IdentityUser>()
    .AddEntityFrameworkStores<AppDbContext>();
```

2. Добавьте строку подключения в `appsettings.json`:
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=FilteringSortingDb;Trusted_Connection=True;"
  }
}
```
### 3. Обновление контроллера для работы с категориями

```csharp
// Controllers/ProductsController.cs
using System.Linq.Dynamic.Core;
using FilteringSortingApi.Data;
using FilteringSortingApi.Models;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

namespace FilteringSortingApi.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class ProductsController : ControllerBase
    {
        private readonly AppDbContext _context;

        public ProductsController(AppDbContext context)
        {
            _context = context;
        }

        [HttpGet]
        public async Task<IActionResult> GetProducts(
            [FromQuery] int? categoryId,
            [FromQuery] string categoryName,
            [FromQuery] decimal? minPrice,
            [FromQuery] decimal? maxPrice,
            [FromQuery] bool? inStock,
            [FromQuery] string sortBy = "Name",
            [FromQuery] string sortOrder = "asc")
        {
            IQueryable<Product> query = _context.Products
                .Include(p => p.Category);

            // Фильтрация по категории
            if (categoryId.HasValue)
            {
                query = query.Where(p => p.CategoryId == categoryId.Value);
            }
            else if (!string.IsNullOrEmpty(categoryName))
            {
                query = query.Where(p => p.Category.Name == categoryName);
            }

            // Фильтрация по цене
            if (minPrice.HasValue)
            {
                query = query.Where(p => p.Price >= minPrice.Value);
            }

            if (maxPrice.HasValue)
            {
                query = query.Where(p => p.Price <= maxPrice.Value);
            }

            // Фильтрация по наличию
            if (inStock.HasValue)
            {
                query = query.Where(p => p.InStock == inStock.Value);
            }

            // Сортировка
            if (!string.IsNullOrEmpty(sortBy))
            {
                string direction = sortOrder.Equals("desc", StringComparison.OrdinalIgnoreCase) ? "desc" : "asc";
                
                // Поддержка сортировки по полям связанной сущности
                if (sortBy.Contains("Category."))
                {
                    query = query.OrderBy($"{sortBy} {direction}");
                }
                else
                {
                    query = query.OrderBy($"p => p.{sortBy} {direction}");
                }
            }

            var products = await query.ToListAsync();
            return Ok(products);
        }
    }
}

// Controllers/CategoriesController.cs
using FilteringSortingApi.Data;
using FilteringSortingApi.Models;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

namespace FilteringSortingApi.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class CategoriesController : ControllerBase
    {
        private readonly AppDbContext _context;

        public CategoriesController(AppDbContext context)
        {
            _context = context;
        }

        [HttpGet]
        public async Task<IActionResult> GetCategories()
        {
            var categories = await _context.Categories.ToListAsync();
            return Ok(categories);
        }
    }
}
```

### 4. Заполнение тестовыми данными

```csharp
// Controllers/SeedController.cs
using FilteringSortingApi.Data;
using FilteringSortingApi.Models;
using Microsoft.AspNetCore.Mvc;

namespace FilteringSortingApi.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class SeedController : ControllerBase
    {
        private readonly AppDbContext _context;

        public SeedController(AppDbContext context)
        {
            _context = context;
        }

        [HttpPost]
        public async Task<IActionResult> SeedData()
        {
            if (_context.Categories.Any() || _context.Products.Any())
            {
                return Ok("Database already has data");
            }

            var categories = new List<Category>
            {
                new Category { Name = "Electronics", Description = "Electronic devices" },
                new Category { Name = "Furniture", Description = "Home and office furniture" },
                new Category { Name = "Stationery", Description = "Office supplies" }
            };

            await _context.Categories.AddRangeAsync(categories);
            await _context.SaveChangesAsync();

            var products = new List<Product>
            {
                new Product { Name = "Laptop", CategoryId = 1, Price = 999.99m, InStock = true },
                new Product { Name = "Smartphone", CategoryId = 1, Price = 699.99m, InStock = true },
                new Product { Name = "Headphones", CategoryId = 1, Price = 149.99m, InStock = false },
                new Product { Name = "Desk Chair", CategoryId = 2, Price = 199.99m, InStock = true },
                new Product { Name = "Coffee Table", CategoryId = 2, Price = 149.99m, InStock = true },
                new Product { Name = "Notebook", CategoryId = 3, Price = 4.99m, InStock = true },
                new Product { Name = "Pen", CategoryId = 3, Price = 1.99m, InStock = false }
            };

            await _context.Products.AddRangeAsync(products);
            await _context.SaveChangesAsync();

            return Ok("Test data seeded successfully");
        }
    }
}
```
1. Создайте миграцию:
```bash
dotnet ef migrations add InitialCreate
```

2. Примените миграцию к базе данных:
```bash
dotnet ef database update
```
### 5. Примеры запросов для тестирования

1. Получить все продукты с категориями:
```
GET /api/products
```

2. Фильтрация по ID категории:
```
GET /api/products?categoryId=1
```

3. Фильтрация по названию категории:
```
GET /api/products?categoryName=Electronics
```

4. Фильтрация по цене и наличию:
```
GET /api/products?minPrice=100&maxPrice=500&inStock=true
```

5. Сортировка по названию категории:
```
GET /api/products?sortBy=Category.Name&sortOrder=asc
```

6. Сортировка по цене (по убыванию):
```
GET /api/products?sortBy=Price&sortOrder=desc
```

7. Заполнить базу тестовыми данными:
```
POST /api/seed
```





