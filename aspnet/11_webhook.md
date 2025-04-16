# Лабораторная работа: C# ASP.NET Core API Webhook с отправкой email

## 1. Создание проекта

### Установка пакетов:
- MailKit
- MimeKit
- Microsoft.EntityFrameworkCore
- Microsoft.EntityFrameworkCore.Design
- Microsoft.EntityFrameworkCore.SqlServer
- Microsoft.EntityFrameworkCore.Tools
- Newtonsoft.Json


### Конфигурация в `appsettings.json`:
```json
{
  "EmailSettings": {
    "SmtpServer": "smtp.yandex.ru",
    "SmtpPort": 587,
    "SmtpUsername": "your@email.com",
    "SmtpPassword": "your-password",
    "FromName": "Product Notifications",
    "FromAddress": "your@email.com"
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=WebhookDb;Trusted_Connection=True;"
  },
}
```
## 1.1 Product
```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public string Category { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}
```
## 1.2 DBContext Data/AppDbContext.cs
```csharp
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Product> Products { get; set; }
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
    }
}
```
## 1.3 ProductRepository в папке Repositories
```csharp
public interface IProductRepository
{
    Task<Product> AddProductAsync(Product product);
    Task<List<Product>> GetAllProductsAsync();
}

public class ProductRepository : IProductRepository
{
    private readonly AppDbContext _context;
    private readonly IWebhookService _webhookService;
    private readonly ILogger<ProductRepository> _logger;

    public ProductRepository(
        AppDbContext context,
        IWebhookService webhookService,
        ILogger<ProductRepository> logger)
    {
        _context = context;
        _webhookService = webhookService;
        _logger = logger;
    }

    public async Task<Product> AddProductAsync(Product product)
    {
        _context.Products.Add(product);
        await _context.SaveChangesAsync();

        // Триггерим webhook после сохранения
        await _webhookService.TriggerProductCreatedAsync(product);

        _logger.LogInformation($"Product {product.Id} added and webhook triggered");

        return product;
    }

    public async Task<List<Product>> GetAllProductsAsync()
    {
        return await _context.Products.ToListAsync();
    }
}
```
## 2. Сервис для отправки email

`Services/EmailService.cs`:
```csharp
using MailKit.Net.Smtp;
using MimeKit;

 public interface IEmailService
 {
     Task SendProductCreatedEmailAsync(Product product, string recipientEmail);
 }

 public class EmailService : IEmailService
 {
     private readonly EmailSettings _emailSettings;
     private readonly ILogger<EmailService> _logger;

     public EmailService(IOptions<EmailSettings> emailSettings, ILogger<EmailService> logger)
     {
         _emailSettings = emailSettings.Value;
         _logger = logger;
     }

     public async Task SendProductCreatedEmailAsync(Product product, string recipientEmail)
     {
         try
         {
             var message = new MimeMessage();
             message.From.Add(new MailboxAddress(_emailSettings.FromName, _emailSettings.FromAddress));
             message.To.Add(new MailboxAddress("", recipientEmail));
             message.Subject = $"Новый продукт: {product.Name}";

             var bodyBuilder = new BodyBuilder
             {
                 HtmlBody = $@"
                 <h1>Добавлен новый продукт</h1>
                 <p><strong>Название:</strong> {product.Name}</p>
                 <p><strong>Цена:</strong> {product.Price:C2}</p>
                 <p><strong>Категория:</strong> {product.Category}</p>
                 <p><strong>Дата добавления:</strong> {product.CreatedAt:dd.MM.yyyy HH:mm}</p>
                 <p><a href='https://your-site.com/products/{product.Id}'>Ссылка на продукт</a></p>"
             };

             message.Body = bodyBuilder.ToMessageBody();

             using var client = new SmtpClient();
             try
             {
                 await client.ConnectAsync(_emailSettings.SmtpServer, _emailSettings.SmtpPort, SecureSocketOptions.StartTls);
                 await client.AuthenticateAsync(_emailSettings.SmtpUsername, _emailSettings.SmtpPassword);
                 await client.SendAsync(message);
                 await client.DisconnectAsync(true);

                 _logger.LogInformation($"Email отправлен для продукта {product.Id} на {recipientEmail}");
                 }
             catch (Exception ex)
             {
                 _logger.LogError($"Ошибка отправки: {ex.Message}");
                 throw;
             }
         }
         catch (Exception ex)
         {
             _logger.LogError(ex, $"Ошибка при отправке email для продукта {product.Id}");
             throw;
         }
     }
 }

 public class EmailSettings
 {
     public string SmtpServer { get; set; }
     public int SmtpPort { get; set; }
     public string SmtpUsername { get; set; }
     public string SmtpPassword { get; set; }
     public string FromName { get; set; }
     public string FromAddress { get; set; }
 }
```

## 3. Webhook сервис

`Services/WebhookService.cs`:
```csharp
public interface IWebhookService
{
    Task TriggerProductCreatedAsync(Product product);
    void SubscribeToProductEvents(string url, string secret, string email = null);
}
public class WebhookService : IWebhookService
{
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly ILogger<WebhookService> _logger;
    private readonly IEmailService _emailService;
    private readonly Dictionary<string, (string Secret, string Email)> _subscribers = new();

    public WebhookService(
        IHttpClientFactory httpClientFactory,
        ILogger<WebhookService> logger,
        IEmailService emailService)
    {
        _httpClientFactory = httpClientFactory;
        _logger = logger;
        _emailService = emailService;
    }

    public async Task TriggerProductCreatedAsync(Product product)
    {
        var payload = new
        {
            EventType = "product.created",
            Timestamp = DateTime.UtcNow.ToString("o"),
            Data = product
        };

        foreach (var subscriber in _subscribers)
        {
            // Отправка HTTP webhook
            await SendHttpWebhookAsync(subscriber.Key, subscriber.Value.Secret, payload);

            // Отправка email
            if (!string.IsNullOrEmpty(subscriber.Value.Email))
            {
                await _emailService.SendProductCreatedEmailAsync(product, subscriber.Value.Email);
            }
        }
    }

    private async Task SendHttpWebhookAsync(string url, string secret, object payload)
    {
        try
        {
            var client = _httpClientFactory.CreateClient();
            var content = new StringContent(
                JsonConvert.SerializeObject(payload),
                Encoding.UTF8,
                "application/json");

            var signature = ComputeSignature(content, secret);
            client.DefaultRequestHeaders.Add("X-Signature", signature);

            var response = await client.PostAsync(url, content);
            response.EnsureSuccessStatusCode();

            _logger.LogInformation($"Webhook отправлен на {url}");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, $"Ошибка при отправке webhook на {url}");
        }
    }

    public void SubscribeToProductEvents(string url, string secret, string email = null)
    {
        _subscribers[url] = (secret, email);
        _logger.LogInformation($"Новый подписчик: {url}, email: {email ?? "не указан"}");
    }

    private string ComputeSignature(StringContent content, string secret)
    {
        using var hmac = new HMACSHA256(Encoding.UTF8.GetBytes(secret));
        var hash = hmac.ComputeHash(content.ReadAsByteArrayAsync().Result);
        return Convert.ToBase64String(hash);
    }
}
```

## 4.1 Контроллер подписки

`Controllers/ProductWebhooksController.cs`:
```csharp
[ApiController]
[Route("api/product-webhooks")]
public class ProductWebhooksController : ControllerBase
{
    private readonly IWebhookService _webhookService;

    public ProductWebhooksController(IWebhookService webhookService)
    {
        _webhookService = webhookService;
    }

    [HttpPost("subscribe")]
    public IActionResult Subscribe([FromBody] ProductWebhookSubscription request)
    {
        if (!Uri.IsWellFormedUriString(request.Url, UriKind.Absolute))
            return BadRequest("Неверный формат URL");

        _webhookService.SubscribeToProductEvents(
            request.Url,
            request.Secret,
            request.Email);

        return Ok(new { message = "Подписка оформлена" });
    }
}

public class ProductWebhookSubscription
{
    public string Url { get; set; }
    public string Secret { get; set; }
    public string Email { get; set; }
}
```
## 4.2 Продукт контроллер
```csharp
[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    private readonly IProductRepository _repository;

    public ProductsController(IProductRepository repository)
    {
        _repository = repository;
    }

    [HttpPost]
    public async Task<ActionResult<Product>> CreateProduct([FromBody] Product product)
    {
        var createdProduct = await _repository.AddProductAsync(product);
        return CreatedAtAction(nameof(GetProduct), new { id = createdProduct.Id }, createdProduct);
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<Product>> GetProduct(int id)
    {
        var product = await _repository.GetAllProductsAsync();
        return product.FirstOrDefault(p => p.Id == id);
    }
}
```
## 5. Регистрация сервисов в Program.cs

```csharp
using AspWebhook.Services;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddHttpClient();

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services.Configure<EmailSettings>(builder.Configuration.GetSection("EmailSettings"));

builder.Services.AddSingleton<IEmailService, EmailService>();
builder.Services.AddSingleton<IWebhookService, WebhookService>();
builder.Services.AddScoped<IProductRepository, ProductRepository>();

builder.Services.AddControllers();

var app = builder.Build();

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```
## 6. Миграции
```bash
dotnet ef migrations add InitialCreate
dotnet ef database update
```
## 7. Пример запроса на подписку с email

POST `/api/product-webhooks/subscribe`
```json
{
  "url": "https://your-server.com/api/webhooks/products",
  "secret": "your-secret-key-123",
  "email": "admin@yourdomain.com"
}
```

## Задание

1. **Шаблоны писем** - вынесите HTML в отдельные файлы. Настройте url-ссылку продкута.
2. Выгрузите проект на GitHub. **Важно** - пароль приложения не должен попасть на GitHub.
