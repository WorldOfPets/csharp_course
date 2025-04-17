# Лабораторная работа: ASP.NET Core API Validation с Entity Framework и DTO

## Цель работы:
Научиться реализовывать валидацию данных в ASP.NET Core Web API с использованием Entity Framework, Data Transfer Objects (DTO) и Fluent Validation.

## Задание:
Создать простое Web API для управления списком продуктов с валидацией входных данных.

## Шаги выполнения:

### 1. Создание проекта

1. Создайте новый проект ASP.NET Core Web API:

2. Добавьте необходимые пакеты:
- Microsoft.EntityFrameworkCore.SqlServer
- Microsoft.EntityFrameworkCore.Tools
- FluentValidation.AspNetCore

### 2. Настройка базы данных и модели

1. Создайте класс `Product` в папке `Models`:
   ```csharp
   namespace ProductApi.Models
   {
       public class Product
       {
           public int Id { get; set; }
           public string Name { get; set; }
           public string Description { get; set; }
           public decimal Price { get; set; }
           public int Stock { get; set; }
           public DateTime CreatedAt { get; set; }
           public DateTime? UpdatedAt { get; set; }
       }
   }
   ```

2. Создайте контекст базы данных `AppDbContext.cs`:
   ```csharp
   using Microsoft.EntityFrameworkCore;

   namespace ProductApi.Models
   {
       public class AppDbContext : DbContext
       {
           public AppDbContext(DbContextOptions<AppDbContext> options) : base(options){}
           public DbSet<Product> Products { get; set; }
       }
   }
   ```

3. Добавьте строку подключения в `appsettings.json`:
   ```json
   "ConnectionStrings": {
     "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=ValidationDB;Trusted_Connection=True;"
   }
   ```

4. Зарегистрируйте контекст в `Program.cs`:
   ```csharp
   builder.Services.AddDbContext<AppDbContext>(options =>
       options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
   ```

5. Создайте и примените миграцию:
   ```bash
   dotnet ef migrations add InitialCreate
   dotnet ef database update
   ```

### 3. Создание DTO и валидаторов

3.1. DTO для DataAnnotations валидации (ProductAnnotationsDto.cs)

   `ProductAnnotationsDto.cs`:
   ```csharp
   using System.ComponentModel.DataAnnotations;

    public class ProductAnnotationsDto
    {
        public int Id { get; set; }
    
        [Required(ErrorMessage = "Name is required")]
        [StringLength(100, MinimumLength = 2, ErrorMessage = "Name must be between 2 and 100 characters")]
        public string Name { get; set; }
    
        [StringLength(500, ErrorMessage = "Description cannot exceed 500 characters")]
        public string Description { get; set; }
    
        [Range(0.01, 999999.99, ErrorMessage = "Price must be between 0.01 and 999999.99")]
        public decimal Price { get; set; }
    
        [Range(0, int.MaxValue, ErrorMessage = "Stock cannot be negative")]
        public int Stock { get; set; }
    }
   ```
3.2. DTO для FluentValidation (ProductFluentDto.cs)
   `ProductFluentDto.cs`:
   ```csharp
   namespace ProductApi.DTOs
   {
      public class ProductFluentDto
      {
          public int Id { get; set; }
          public string Name { get; set; }
          public string Description { get; set; }
          public decimal Price { get; set; }
          public int Stock { get; set; }
      }
   }
   ```
3.3. DTO для ручной валидации (ProductManualDto.cs)
   `ProductManualDto.cs` (для ответов):
   ```csharp
   namespace ProductApi.DTOs
   {
       public class ProductManualDto
        {
            public int Id { get; set; }
            public string Name { get; set; }
            public string Description { get; set; }
            public decimal Price { get; set; }
            public int Stock { get; set; }
        }
   }
   ```

4 Создайте папку `Validators` и добавьте валидаторы:

4.1 `ProductFluentDtoValidator.cs`:
   ```csharp
   using FluentValidation;
    using ProductValidationApi.DTOs;
    
    public class ProductFluentDtoValidator : AbstractValidator<ProductFluentDto>
    {
        public ProductFluentDtoValidator()
        {
            RuleFor(x => x.Name)
                .NotEmpty().WithMessage("Name is required")
                .Length(2, 100).WithMessage("Name must be between 2 and 100 characters");
    
            RuleFor(x => x.Description)
                .MaximumLength(500).WithMessage("Description cannot exceed 500 characters");
    
            RuleFor(x => x.Price)
                .GreaterThan(0).WithMessage("Price must be greater than 0")
                .LessThan(1000000).WithMessage("Price must be less than 1,000,000");
    
            RuleFor(x => x.Stock)
                .GreaterThanOrEqualTo(0).WithMessage("Stock cannot be negative");
        }
    }
   ```
4.2 Ручная валидация
Создайте сервис для ручной валидации (Services/ManualValidatorService.cs):
   `ManualValidatorService.cs`:
   ```csharp
   using ProductValidationApi.DTOs;

    public class ManualValidatorService
    {
        public (bool IsValid, List<string> Errors) ValidateProduct(ProductManualDto product)
        {
            var errors = new List<string>();
    
            if (string.IsNullOrEmpty(product.Name) || product.Name.Length < 2 || product.Name.Length > 100)
                errors.Add("Name must be between 2 and 100 characters");
    
            if (product.Description?.Length > 500)
                errors.Add("Description cannot exceed 500 characters");
    
            if (product.Price <= 0 || product.Price >= 1000000)
                errors.Add("Price must be between 0 and 1,000,000");
    
            if (product.Stock < 0)
                errors.Add("Stock cannot be negative");
    
            return (!errors.Any(), errors);
        }
    }
   ```

4.3 Зарегистрируйте валидаторы в `Program.cs`:
   ```csharp
    //fluent
   builder.Services.AddFluentValidationAutoValidation();
   builder.Services.AddValidatorsFromAssemblyContaining<ProductCreateDtoValidator>();
    //manual
    builder.Services.AddScoped<ManualValidatorService>();
   ```

### 4. Создание контроллера

Создайте `ProductsController.cs`:
```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using ProductValidationApi.DTOs;
using ProductValidationApi.Models;
using ProductValidationApi.Services;
using ProductValidationApi.Validators;

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly AppDbContext _context;
    private readonly ManualValidatorService _manualValidator;

    public ProductsController(AppDbContext context, ManualValidatorService manualValidator)
    {
        _context = context;
        _manualValidator = manualValidator;
    }

    // 1. Валидация через DataAnnotations
    [HttpPost("annotations")]
    public IActionResult CreateWithAnnotations([FromBody] ProductAnnotationsDto productDto)
    {
        if (!ModelState.IsValid)
        {
            return BadRequest(ModelState);
        }

        // Далее обработка валидных данных...
        return Ok("Product created with DataAnnotations validation");
    }

    // 2. Валидация через FluentValidation
    [HttpPost("fluent")]
    public IActionResult CreateWithFluent([FromBody] ProductFluentDto productDto)
    {
        // Валидация выполняется автоматически перед вызовом метода
        if (!ModelState.IsValid)
        {
            return BadRequest(ModelState);
        }

        // Далее обработка валидных данных...
        return Ok("Product created with FluentValidation");
    }

    // 3. Ручная валидация
    [HttpPost("manual")]
    public IActionResult CreateWithManualValidation([FromBody] ProductManualDto productDto)
    {
        var (isValid, errors) = _manualValidator.ValidateProduct(productDto);

        if (!isValid)
        {
            return BadRequest(new { Errors = errors });
        }

        // Далее обработка валидных данных...
        return Ok("Product created with manual validation");
    }

    // Другие CRUD методы...
}
```

### 5. Тестирование API

1. Запустите приложение:

2. Используйте Postman для тестирования:
   - Попробуйте создать продукт с невалидными данными (например, отрицательной ценой или пустым именем) и убедитесь, что получаете соответствующие сообщения об ошибках.

## Задания для самостоятельной работы:

1. Реализуйте другие методы GET, GET ALL, PUT, DELETE (Создайте DTO объекты для этих запросов если необходимо).
2. Добавьте поле manufacturer (изготовитель/производитель) для продукта испльзуйте Regex (регулярное выражение, чтобы производитель соответствовал следующему паттерну ООО "ИмяПроизводителяБезПробелов"
3. Выгрузите проект на GitHub.
