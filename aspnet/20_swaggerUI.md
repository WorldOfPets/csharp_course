# Лабораторная работа: Создание Web API с использованием ASP.NET Core и Swagger UI

## Цель работы
Научиться создавать RESTful API с использованием ASP.NET Core и документировать его с помощью Swagger UI.

## 1. Создание проекта

### 1.1. Создайте новый проект ASP.NET Core Web API:

### 1.2. Убедитесь, что в проекте есть необходимые пакеты:
- Swashbuckle.AspNetCore

## 2. Настройка Swagger

### 2.1. В файле Program.cs добавьте сервисы Swagger:
```csharp
using Microsoft.OpenApi.Models;
using System.Reflection;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "BookStore API",
        Version = "v1",
        Description = "API для управления книжным магазином",
        Contact = new OpenApiContact
        {
            Name = "Разработчик",
            Email = "dev@example.com"
        }
    });

    // Включение XML-комментариев
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    c.IncludeXmlComments(xmlPath);
});
var app = builder.Build();
app.UseRouting();
app.UseEndpoints(endpoints =>
{
    endpoints.MapControllers();
});
app.UseSwagger();
app.UseSwaggerUI();

app.Run();
```

## 3. Создание модели данных

### 3.1. Создайте папку Models и добавьте класс Book:
```csharp
public class Book
{
    public int Id { get; set; }
    
    [Required]
    public string Title { get; set; }
    
    public string Author { get; set; }
    
    [Range(0, 1000)]
    public decimal Price { get; set; }
    
    public DateTime PublishDate { get; set; }
}
```

## 4. Создание контроллера

### 4.1. Создайте контроллер BooksController:
```csharp
[ApiController]
[Route("api/[controller]")]
[Produces("application/json")]
public class BooksController : ControllerBase
{
    private static List<Book> _books = new()
    {
        new Book { Id = 1, Title = "Война и мир", Author = "Лев Толстой", Price = 500, PublishDate = new DateTime(1869, 1, 1) },
        new Book { Id = 2, Title = "Преступление и наказание", Author = "Фёдор Достоевский", Price = 450, PublishDate = new DateTime(1866, 1, 1) }
    };

    /// <summary>
    /// Получить список всех книг
    /// </summary>
    /// <returns>Список книг</returns>
    [HttpGet]
    [ProducesResponseType(StatusCodes.Status200OK)]
    public ActionResult<IEnumerable<Book>> GetAllBooks()
    {
        return Ok(_books);
    }

    /// <summary>
    /// Получить книгу по ID
    /// </summary>
    /// <param name="id">Идентификатор книги</param>
    /// <returns>Книга</returns>
    [HttpGet("{id}")]
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public ActionResult<Book> GetBook(int id)
    {
        var book = _books.FirstOrDefault(b => b.Id == id);
        if (book == null)
        {
            return NotFound();
        }
        return Ok(book);
    }

    /// <summary>
    /// Добавить новую книгу
    /// </summary>
    /// <param name="book">Данные новой книги</param>
    /// <returns>Созданная книга</returns>
    [HttpPost]
    [ProducesResponseType(StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public ActionResult<Book> CreateBook([FromBody] Book book)
    {
        if (!ModelState.IsValid)
        {
            return BadRequest(ModelState);
        }
        
        book.Id = _books.Max(b => b.Id) + 1;
        _books.Add(book);
        
        return CreatedAtAction(nameof(GetBook), new { id = book.Id }, book);
    }

    /// <summary>
    /// Обновить существующую книгу
    /// </summary>
    /// <param name="id">Идентификатор книги</param>
    /// <param name="book">Обновленные данные книги</param>
    /// <returns></returns>
    [HttpPut("{id}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public IActionResult UpdateBook(int id, [FromBody] Book book)
    {
        if (id != book.Id)
        {
            return BadRequest();
        }
        
        var existingBook = _books.FirstOrDefault(b => b.Id == id);
        if (existingBook == null)
        {
            return NotFound();
        }
        
        existingBook.Title = book.Title;
        existingBook.Author = book.Author;
        existingBook.Price = book.Price;
        existingBook.PublishDate = book.PublishDate;
        
        return NoContent();
    }

    /// <summary>
    /// Удалить книгу
    /// </summary>
    /// <param name="id">Идентификатор книги</param>
    /// <returns></returns>
    [HttpDelete("{id}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public IActionResult DeleteBook(int id)
    {
        var book = _books.FirstOrDefault(b => b.Id == id);
        if (book == null)
        {
            return NotFound();
        }
        
        _books.Remove(book);
        return NoContent();
    }
}
```

## 5. Включение XML-документации

### 5.1. В файле проекта (.csproj) добавьте:
```xml
<PropertyGroup>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>
  <NoWarn>$(NoWarn);1591</NoWarn>
</PropertyGroup>
```

## 6. Запуск и тестирование API

### 6.1. Запустите приложение:
```bash
dotnet run
```

### 6.2. Откройте Swagger UI в браузере:
```
http://localhost:<port>/swagger
```
