# Базовая структура проекта
```
MyApiProject/
├── publish/                           # Build вашего приложения
|   ├── bin/
|   └── obj/
├── src/
│   ├── MyApiProject/                  # Основной проект API
│   │   ├── Controllers/               # Контроллеры API
│   │   ├── Models/                    # Модели данных (DTOs, Entities)
│   │   ├── Services/                  # Бизнес-логика и сервисы
│   │   ├── Repositories/              # Доступ к данным (если используется)
│   │   ├── Interfaces/                # Интерфейсы для сервисов и репозиториев
│   │   ├── Entities/                  # Сущности базы данных (EF Core) - чаще расположены в Models
│   │   ├── Mappings/                  # Профили AutoMapper
│   │   ├── Middleware/                # Пользовательские middleware
│   │   ├── Filters/                   # Фильтры API
│   │   ├── Validators/                # Валидаторы (FluentValidation)
│   │   ├── Extensions/                # Методы расширения
│   │   ├── Configuration/             # Конфигурационные классы
│   │   ├── Properties/                # Свойства проекта (launchSettings.json)
│   │   ├── Program.cs                 # Точка входа
│   │   └── appsettings.json           # Конфигурационные файлы
│   └── MyApiProject.Tests/            # Юнит-тесты
├── docs/                              # Документация
├── .gitignore                         # Игнорируемые файлы Git
|__ MyApiProject.sln                   # .sln файл проекта
└── README.md                          # Описание проекта
```
# Лабораторная работа: ASP.NET Core RESTful API - Unit Tests

## Цель работы
Научиться создавать unit-тесты для ASP.NET Core RESTful API, покрывающие основные CRUD операции и бизнес-логику приложения.

## Часть 1: Создание простого RESTful API с базой данных

### 1. Создайте новый проект ASP.NET Core Web API

### 2. Добавьте необходимые пакеты

- Microsoft.EntityFrameworkCore
- Microsoft.EntityFrameworkCore.Design
- Microsoft.EntityFrameworkCore.SqlServer
- Microsoft.EntityFrameworkCore.Tools

### 3. Создайте модель Product

`Models/Product.cs`:
```csharp
namespace ProductApi.Models;

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int Stock { get; set; }
    public string Description { get; set; } = string.Empty;
}
```

### 4. Создайте DbContext

`Data/ProductContext.cs`:
```csharp
using Microsoft.EntityFrameworkCore;
using ProductApi.Models;

namespace ProductApi.Data;

public class ProductContext : DbContext
{
    public ProductContext(DbContextOptions<ProductContext> options) : base(options)
    {
    }

    public virtual DbSet<Product> Products { get; set; }
}
```

### 5. Настройте базу данных в Program.cs

Добавьте в `Program.cs`:
```csharp
// Добавьте после builder.Services.AddControllers();
builder.Services.AddDbContext<ProductContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
```

Добавьте в `appsettings.json`:
```json
"ConnectionStrings": {
  "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=ProductApiDB;Trusted_Connection=True;"
}
```

### 6. Создайте контроллер ProductsController

`Controllers/ProductsController.cs`:
```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using ProductApi.Data;
using ProductApi.Models;

namespace ProductApi.Controllers;

[Route("api/[controller]")]
[ApiController]
public class ProductsController : ControllerBase
{
    private readonly ProductContext _context;

    public ProductsController(ProductContext context)
    {
        _context = context;
    }

    // GET: api/Products
    [HttpGet]
    public async Task<ActionResult<IEnumerable<Product>>> GetProducts()
    {
        return await _context.Products.ToListAsync();
    }

    // GET: api/Products/5
    [HttpGet("{id}")]
    public async Task<ActionResult<Product>> GetProduct(int id)
    {
        var product = await _context.Products.FindAsync(id);

        if (product == null)
        {
            return NotFound();
        }

        return product;
    }

    // PUT: api/Products/5
    [HttpPut("{id}")]
    public async Task<IActionResult> PutProduct(int id, Product product)
    {
        if (id != product.Id)
        {
            return BadRequest();
        }

        _context.Entry(product).State = EntityState.Modified;

        try
        {
            await _context.SaveChangesAsync();
        }
        catch (DbUpdateConcurrencyException)
        {
            if (!ProductExists(id))
            {
                return NotFound();
            }
            else
            {
                throw;
            }
        }

        return NoContent();
    }

    // POST: api/Products
    [HttpPost]
    public async Task<ActionResult<Product>> PostProduct(Product product)
    {
        await _context.Products.AddAsync(product);
        await _context.SaveChangesAsync();

        return CreatedAtAction("GetProduct", new { id = product.Id }, product);
    }

    // DELETE: api/Products/5
    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteProduct(int id)
    {
        var product = await _context.Products.FindAsync(id);
        if (product == null)
        {
            return NotFound();
        }

        _context.Products.Remove(product);
        await _context.SaveChangesAsync();

        return NoContent();
    }

    private bool ProductExists(int id)
    {
        return _context.Products.Any(e => e.Id == id);
    }
}
```

## Часть 2: Создание Unit-тестов для API

### 1. Создайте тестовый проект xUnit

```bash
dotnet new xunit -n ProductApi.Tests
cd ProductApi.Tests
dotnet add reference ../ProductApi/ProductApi.csproj
dotnet add package Microsoft.EntityFrameworkCore.InMemory
dotnet add package Moq
dotnet add package FluentAssertions
dotnet add package MockQueryable.Moq
dotnet add package xunit
dotnet add package xunit.runner.visualstudio
dotnet add package Microsoft.NET.Test.Sdk
dotnet add package coverlet.collector
```

### 2. Создайте тесты для контроллера

`ProductsControllerTests.cs`:
```csharp
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using FluentAssertions;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using ProductApi.Controllers;
using ProductApi.Data;
using ProductApi.Models;
using Xunit;

namespace ProductApi.Tests.Controllers;

public class ProductsControllerTests
{
    private async Task<ProductContext> GetDatabaseContext()
    {
        var options = new DbContextOptionsBuilder<ProductContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
            .Options;

        var databaseContext = new ProductContext(options);
        await databaseContext.Database.EnsureCreatedAsync();

        if (!await databaseContext.Products.AnyAsync())
        {
            await databaseContext.Products.AddRangeAsync(GetTestProducts());
            await databaseContext.SaveChangesAsync();
        }

        return databaseContext;
    }

    private List<Product> GetTestProducts()
    {
        return new List<Product>
    {
        new() { Id = 1, Name = "Test Product 1", Price = 10.99m, Stock = 100, Description = "Description 1" },
        new() { Id = 2, Name = "Test Product 2", Price = 20.99m, Stock = 200, Description = "Description 2" },
        new() { Id = 3, Name = "Test Product 3", Price = 30.99m, Stock = 300, Description = "Description 3" }
    };
    }

    [Fact]
    public async Task GetProducts_ReturnsAllProducts()
    {
        // Arrange
        var dbContext = await GetDatabaseContext();
        var controller = new ProductsController(dbContext);

        // Act
        var result = await controller.GetProducts();

        // Assert
        var actionResult = Assert.IsType<ActionResult<IEnumerable<Product>>>(result);
        var returnValue = Assert.IsType<List<Product>>(actionResult.Value);
        returnValue.Should().HaveCount(3);
    }

    [Fact]
    public async Task GetProduct_WithValidId_ReturnsProduct()
    {
        // Arrange
        var dbContext = await GetDatabaseContext();
        var controller = new ProductsController(dbContext);
        var testId = 1;

        // Act
        var result = await controller.GetProduct(testId);

        // Assert
        var actionResult = Assert.IsType<ActionResult<Product>>(result);
        var product = Assert.IsType<Product>(actionResult.Value);
        Assert.Equal(testId, product.Id);
    }

    [Fact]
    public async Task GetProduct_WithInvalidId_ReturnsNotFound()
    {
        // Arrange
        var dbContext = await GetDatabaseContext();
        var controller = new ProductsController(dbContext);
        var testId = 999;

        // Act
        var result = await controller.GetProduct(testId);

        // Assert
        Assert.IsType<NotFoundResult>(result.Result);
    }

    [Fact]
    public async Task PostProduct_WithValidProduct_ReturnsCreatedAtAction()
    {
        // Arrange
        var dbContext = await GetDatabaseContext();
        var controller = new ProductsController(dbContext);
        var newProduct = new Product
        {
            Name = "New Product",
            Price = 40.99m,
            Stock = 400,
            Description = "New Description"
        };

        // Act
        var result = await controller.PostProduct(newProduct);

        // Assert
        var actionResult = Assert.IsType<ActionResult<Product>>(result);
        var createdAtActionResult = Assert.IsType<CreatedAtActionResult>(actionResult.Result);
        var returnedProduct = Assert.IsType<Product>(createdAtActionResult.Value);

        Assert.Equal(newProduct.Name, returnedProduct.Name);
        Assert.Equal(newProduct.Price, returnedProduct.Price);
    }

    [Fact]
    public async Task PutProduct_WithValidIdAndProduct_ReturnsNoContent()
    {

        // Arrange
        var dbContext = await GetDatabaseContext();
        var controller = new ProductsController(dbContext);
        // Clear the tracker to avoid conflicts
        dbContext.ChangeTracker.Clear();
        var testId = 1;
        var updatedProduct = new Product
        {
            Id = testId,
            Name = "Updated Product",
            Price = 15.99m,
            Stock = 150,
            Description = "Updated Description"
        };
        
        // Act
        var result = await controller.PutProduct(testId, updatedProduct);

        // Assert
        Assert.IsType<NoContentResult>(result);

        // Verify the update
        var product = await dbContext.Products.FindAsync(testId);
        Assert.Equal(updatedProduct.Name, product.Name);
        Assert.Equal(updatedProduct.Price, product.Price);
    }

    [Fact]
    public async Task PutProduct_WithInvalidId_ReturnsBadRequest()
    {
        // Arrange
        var dbContext = await GetDatabaseContext();
        var controller = new ProductsController(dbContext);
        var testId = 1;
        var updatedProduct = new Product
        {
            Id = 2, // Different ID
            Name = "Updated Product",
            Price = 15.99m,
            Stock = 150,
            Description = "Updated Description"
        };

        // Act
        var result = await controller.PutProduct(testId, updatedProduct);

        // Assert
        Assert.IsType<BadRequestResult>(result);
    }

    [Fact]
    public async Task DeleteProduct_WithValidId_ReturnsNoContent()
    {
        // Arrange
        var dbContext = await GetDatabaseContext();
        var controller = new ProductsController(dbContext);
        var testId = 1;

        // Act
        var result = await controller.DeleteProduct(testId);

        // Assert
        Assert.IsType<NoContentResult>(result);

        // Verify the deletion
        var product = await dbContext.Products.FindAsync(testId);
        Assert.Null(product);
    }

    [Fact]
    public async Task DeleteProduct_WithInvalidId_ReturnsNotFound()
    {
        // Arrange
        var dbContext = await GetDatabaseContext();
        var controller = new ProductsController(dbContext);
        var testId = 999;

        // Act
        var result = await controller.DeleteProduct(testId);

        // Assert
        Assert.IsType<NotFoundResult>(result);
    }
}
```
### 2.1 Moq тесты для контроллера
`ProductsControllerTestsWithMoq.cs`:
```csharp
public class ProductsControllerTestsWithMoq
{
    private readonly Mock<ProductContext> _mockContext;
    private readonly ProductsController _controller;
    private readonly List<Product> _testProducts;

    public ProductsControllerTestsWithMoq()
    {
        var options = new DbContextOptionsBuilder<ProductContext>()
            .UseInMemoryDatabase(databaseName: "TestDb_GetProducts")
            .Options;
        //var options = new DbContextOptionsBuilder<ProductContext>().Options;
        _mockContext = new Mock<ProductContext>(options) { CallBase = true };
        _testProducts = new List<Product>
        {
            new() { Id = 1, Name = "Test Product 1", Price = 10.99m, Stock = 100, Description = "Description 1" },
            new() { Id = 2, Name = "Test Product 2", Price = 20.99m, Stock = 200, Description = "Description 2" },
            new() { Id = 3, Name = "Test Product 3", Price = 30.99m, Stock = 300, Description = "Description 3" }
        };

        // Мокаем DbSet<Product>
        var mockSet = _testProducts.AsQueryable().BuildMockDbSet();
        // Мокаем ProductContext
        //_mockContext = new Mock<ProductContext>();
        _mockContext.Setup(c => c.Products).Returns(mockSet.Object);
        _mockContext.Setup(c => c.SaveChangesAsync(It.IsAny<CancellationToken>())).ReturnsAsync(1);

        _controller = new ProductsController(_mockContext.Object);
    }

    [Fact]
    public async Task GetProducts_ReturnsAllProducts()
    {
        // Act
        var result = await _controller.GetProducts();

        // Assert
        var actionResult = Assert.IsType<ActionResult<IEnumerable<Product>>>(result);
        var returnValue = Assert.IsType<List<Product>>(actionResult.Value);
        returnValue.Should().HaveCount(3);
        _mockContext.Verify(m => m.Products, Times.Once);
    }

    [Fact]
    public async Task GetProduct_WithValidId_ReturnsProduct()
    {
        // Arrange
        var testId = 1;
        _mockContext.Setup(m => m.Products.FindAsync(testId))
            .ReturnsAsync(_testProducts.First(p => p.Id == testId));

        // Act
        var result = await _controller.GetProduct(testId);

        // Assert
        var actionResult = Assert.IsType<ActionResult<Product>>(result);
        var product = Assert.IsType<Product>(actionResult.Value);
        product.Id.Should().Be(testId);
        _mockContext.Verify(m => m.Products.FindAsync(testId), Times.Once);
    }

    [Fact]
    public async Task PostProduct_AddsNewProduct()
    {
        // Arrange
        var newProduct = new Product
        {
            Name = "New Product",
            Price = 40.99m,
            Stock = 400,
            Description = "New Description"
        };

        var mockSet = new Mock<DbSet<Product>>();
        _mockContext.Setup(m => m.Products).Returns(mockSet.Object);
        _mockContext.Setup(m => m.SaveChangesAsync(It.IsAny<CancellationToken>())).ReturnsAsync(1);

        // Act
        var result = await _controller.PostProduct(newProduct);

        // Assert
        var actionResult = Assert.IsType<ActionResult<Product>>(result);
        var createdAtActionResult = Assert.IsType<CreatedAtActionResult>(actionResult.Result);
        Assert.Equal(newProduct, createdAtActionResult.Value);

        mockSet.Verify(m => m.AddAsync(newProduct, It.IsAny<CancellationToken>()), Times.Once);
        _mockContext.Verify(m => m.SaveChangesAsync(It.IsAny<CancellationToken>()), Times.Once);
    }

    [Fact]
    public async Task DeleteProduct_RemovesProduct()
    {
        // Arrange
        var testId = 1;
        var productToRemove = _testProducts.First(p => p.Id == testId);

        _mockContext.Setup(m => m.Products.FindAsync(testId))
            .ReturnsAsync(productToRemove);

        // Act
        var result = await _controller.DeleteProduct(testId);

        // Assert
        Assert.IsType<NoContentResult>(result);
        _mockContext.Verify(m => m.Products.Remove(productToRemove), Times.Once);
        _mockContext.Verify(m => m.SaveChangesAsync(It.IsAny<CancellationToken>()), Times.Once);
    }
}
```
### 3. Запустите тесты

## Задание для самостоятельной работы

1. После запуска методов некоторые части кода в контроллере будут помечены красным цветом. Это та область которая не покрыта тестами - исправьте это.
2. Загрузите работу на GitHub.

