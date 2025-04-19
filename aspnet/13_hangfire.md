# Лабораторная работа: "Реализация фоновых задач в ASP.NET Core с использованием Hangfire"

## Цель работы
Научиться интегрировать и использовать Hangfire для выполнения фоновых задач, отложенных и периодических заданий в ASP.NET Core REST API.

## Теоретическая часть

### Основные понятия
1. **Hangfire** - библиотека для выполнения фоновых задач в .NET приложениях
2. **Фоновая задача** - операция, выполняемая вне контекста HTTP-запроса
3. **Отложенная задача** - задача, выполняемая однократно в указанное время
4. **Периодическая задача** - задача, выполняемая по расписанию
5. **Dashboard** - веб-интерфейс для мониторинга задач

## Практическая часть

### Задание 1: Создание проекта и настройка Hangfire

1. Создайте новый проект ASP.NET Core Web API:

2. Установите необходимые пакеты:
```bash
dotnet add package Hangfire.Core
dotnet add package Hangfire.AspNetCore
dotnet add package Hangfire.MemoryStorage
```

3. Настройте Hangfire в `Program.cs`:
```csharp
using Hangfire;

var builder = WebApplication.CreateBuilder(args);

// Добавление сервисов Hangfire
builder.Services.AddHangfire(config => 
    config.UseMemoryStorage());
builder.Services.AddHangfireServer();

var app = builder.Build();

// Настройка dashboard
app.UseHangfireDashboard();

app.MapControllers();
app.Run();
```

### Задание 2: Создание сервиса для фоновых задач

1. Создайте сервис `EmailService.cs`:
```csharp
public class EmailService
{
    private readonly ILogger<EmailService> _logger;

    public EmailService(ILogger<EmailService> logger)
    {
        _logger = logger;
    }

    public void SendWelcomeEmail(string email, string username)
    {
        _logger.LogInformation($"Sending welcome email to {email} for {username}");
        // Имитация отправки email
        Thread.Sleep(5000);
        _logger.LogInformation($"Email sent to {email}");
    }

    public void SendDailyReport()
    {
        _logger.LogInformation("Sending daily report...");
        // Имитация генерации и отправки отчета
        Thread.Sleep(3000);
        _logger.LogInformation("Daily report sent");
    }
}
```

2. Зарегистрируйте сервис в `Program.cs`:
```csharp
builder.Services.AddScoped<EmailService>();
```

### Задание 3: Создание контроллера для управления задачами

Создайте `JobsController.cs`:
```csharp
[ApiController]
[Route("api/[controller]")]
public class JobsController : ControllerBase
{
    private readonly IBackgroundJobClient _backgroundJob;
    private readonly IRecurringJobManager _recurringJob;
    private readonly EmailService _emailService;

    public JobsController(
        IBackgroundJobClient backgroundJob,
        IRecurringJobManager recurringJob,
        EmailService emailService)
    {
        _backgroundJob = backgroundJob;
        _recurringJob = recurringJob;
        _emailService = emailService;
    }

    [HttpPost("welcome-email")]
    public IActionResult ScheduleWelcomeEmail([FromBody] WelcomeEmailRequest request)
    {
        var jobId = _backgroundJob.Enqueue(() => 
            _emailService.SendWelcomeEmail(request.Email, request.Username));
        
        return Ok(new { JobId = jobId });
    }

    [HttpPost("delayed-email")]
    public IActionResult ScheduleDelayedEmail([FromBody] WelcomeEmailRequest request)
    {
        var jobId = _backgroundJob.Schedule(() => 
            _emailService.SendWelcomeEmail(request.Email, request.Username),
            TimeSpan.FromMinutes(5));
        
        return Ok(new { JobId = jobId });
    }

    [HttpPost("daily-report")]
    public IActionResult StartDailyReport()
    {
        _recurringJob.AddOrUpdate(
            "daily-report-job",
            () => _emailService.SendDailyReport(),
            Cron.Daily);
        
        return Ok(new { Message = "Daily report job scheduled" });
    }

    [HttpGet("status/{jobId}")]
    public IActionResult GetJobStatus(string jobId)
    {
        var connection = JobStorage.Current.GetConnection();
        var jobData = connection.GetJobData(jobId);
        
        if (jobData == null)
            return NotFound();
            
        return Ok(new { 
            jobData.Job,
            jobData.State,
            jobData.CreatedAt,
            jobData.ExpireAt
        });
    }
}

public record WelcomeEmailRequest(string Email, string Username);
```

### Задание 4: Тестирование работы Hangfire

1. Запустите приложение:

2. Откройте Hangfire Dashboard по адресу: `http://localhost:PORT/hangfire`

3. Протестируйте API endpoints:
   - `POST /api/jobs/welcome-email` - немедленная отправка email
   - `POST /api/jobs/delayed-email` - отложенная отправка email
   - `POST /api/jobs/daily-report` - настройка ежедневного отчета
   - `GET /api/jobs/status/{jobId}` - проверка статуса задачи

### Задание 5: Интеграция с базой данных (дополнительно)

1. Обновите конфигурацию в `Program.cs`:
```csharp
var connectionString = builder.Configuration.GetConnectionString("HangfireConnection");
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));
// Добавление сервисов Hangfire
builder.Services.AddHangfire(config =>
    config.UseSqlServerStorage(connectionString,
    new SqlServerStorageOptions
    {
        PrepareSchemaIfNecessary = true, // Автоматически создает таблицы
        SchemaName = "Hangfire"
    }));
```
2. Добавьте `AppDbContext` в `Data`
```csharp
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options)
    {
    }
}
```
4. Добавьте строку подключения в `appsettings.json`:
```json
{
  "ConnectionStrings": {
    "HangfireConnection": "Server=(localdb)\\mssqllocaldb;Database=HangfireTest;Trusted_Connection=True;"
  }
}
```

## Дополнительные задания

1. Реализуйте отмену периодической задачи
2. Выгрузите проект на GitHub.
3. Интегрируйте Hangfire с существующим сервисом в вашем приложении (пусть приложение Hangfire использует другой DbContext отлчиный от основного. В лучшем варианте если он будет как отдельный проект а общение между ними будет через WebHook`и)
