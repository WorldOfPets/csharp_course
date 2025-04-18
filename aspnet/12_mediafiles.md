# Лабораторная работа: Разработка RESTful API для работы с медиафайлами в ASP.NET Core

## Цель работы
Научиться создавать RESTful API для загрузки, хранения и управления медиафайлами с использованием ASP.NET Core.

## Теоретическая часть

### Основные концепции
1. **IFormFile** - интерфейс в ASP.NET Core для работы с загружаемыми файлами
2. **MIME-типы** - идентификаторы типа контента (image/jpeg, application/pdf и т.д.)
3. **Потоковая передача файлов** - эффективный способ работы с большими файлами

## Практическая часть

### Задание 1: Создание проекта
1. Создайте новый проект ASP.NET Core Web API
    - Microsoft.EntityFrameworkCore
    - Microsoft.EntityFrameworkCore.SqlServer
    - Microsoft.EntityFrameworkCore.Tools

### Задание 2: Настройка модели и контекста
1. Создайте модель `MediaFile`:
```csharp
public class MediaFile
{
    public int Id { get; set; }
    public string FileName { get; set; }
    public string OriginalName { get; set; }
    public string ContentType { get; set; }
    public long Size { get; set; }
    public DateTime UploadDate { get; set; }
    public string Description { get; set; }
    public string FileUrl { get; set; }
}
```

2. Создайте контекст базы данных:
```csharp
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }
    
    public DbSet<MediaFile> MediaFiles { get; set; }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<MediaFile>()
            .Property(m => m.UploadDate)
            .HasDefaultValueSql("GETDATE()");
    }
}
```

### Задание 3: Реализация контроллера
Создайте `MediaFilesController.cs` с методами:

```csharp
[ApiController]
[Route("api/[controller]")]
public class MediaFilesController : ControllerBase
{
    private readonly AppDbContext _context;
    private readonly IWebHostEnvironment _env;
    private readonly IHttpContextAccessor _httpContextAccessor;

    public MediaFilesController(AppDbContext context, IWebHostEnvironment env, IHttpContextAccessor httpContextAccessor)
    {
        _context = context;
        _env = env;
        _httpContextAccessor = httpContextAccessor;
    }

    private string GenerateFileUrl(string fileName)
    {
        var request = _httpContextAccessor.HttpContext.Request;
        return $"{request.Scheme}://{request.Host}/uploads/{fileName}";
    }

    // POST api/mediafiles/upload
    [HttpPost("upload")]
    public async Task<ActionResult<MediaFile>> UploadFile(
        IFormFile file, 
        [FromForm] string description = null)
    {
        // Валидация файла
        if (file == null || file.Length == 0)
            return BadRequest("Файл не выбран");

        if (file.Length > 10 * 1024 * 1024) // 10MB
            return BadRequest("Файл слишком большой");

        var allowedExtensions = new[] { ".jpg", ".jpeg", ".png", ".gif", ".pdf", ".mp4" };
        var fileExtension = Path.GetExtension(file.FileName).ToLower();
        if (!allowedExtensions.Contains(fileExtension))
            return BadRequest("Недопустимый формат файла");

        // Сохранение файла
        var uploadsFolder = Path.Combine(_env.WebRootPath, "uploads");
        if (!Directory.Exists(uploadsFolder))
            Directory.CreateDirectory(uploadsFolder);

        var uniqueFileName = Guid.NewGuid().ToString() + fileExtension;
        var filePath = Path.Combine(uploadsFolder, uniqueFileName);

        using (var stream = new FileStream(filePath, FileMode.Create))
        {
            await file.CopyToAsync(stream);
        }

        // Сохранение в БД
        var mediaFile = new MediaFile
        {
            FileName = uniqueFileName,
            OriginalName = file.FileName,
            ContentType = file.ContentType,
            Size = file.Length,
            Description = description,
            FileUrl = GenerateFileUrl(uniqueFileName)
        };

        _context.MediaFiles.Add(mediaFile);
        await _context.SaveChangesAsync();

        return CreatedAtAction(nameof(GetFile), new { id = mediaFile.Id }, mediaFile);
    }

    // GET api/mediafiles/{id}
    [HttpGet("{id}")]
    public async Task<ActionResult<MediaFile>> GetFile(int id)
    {
        var file = await _context.MediaFiles.FindAsync(id);
        if (file == null)
            return NotFound();

        return file;
    }

    // GET api/mediafiles/{id}/download
    [HttpGet("{id}/download")]
    public async Task<IActionResult> DownloadFile(int id)
    {
        var file = await _context.MediaFiles.FindAsync(id);
        if (file == null)
            return NotFound();

        var filePath = Path.Combine(_env.WebRootPath, "uploads", file.FileName);
        if (!System.IO.File.Exists(filePath))
            return NotFound();

        return PhysicalFile(filePath, file.ContentType, file.OriginalName);
    }

    // DELETE api/mediafiles/{id}
    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteFile(int id)
    {
        var file = await _context.MediaFiles.FindAsync(id);
        if (file == null)
            return NotFound();

        var filePath = Path.Combine(_env.WebRootPath, "uploads", file.FileName);
        if (System.IO.File.Exists(filePath))
            System.IO.File.Delete(filePath);

        _context.MediaFiles.Remove(file);
        await _context.SaveChangesAsync();

        return NoContent();
    }

    // GET api/mediafiles
    [HttpGet]
    public async Task<ActionResult<IEnumerable<MediaFile>>> GetAllFiles()
    {
        return await _context.MediaFiles.ToListAsync();
    }
}
```

### Задание 4: Настройка приложения
В `Program.cs` добавьте:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Добавьте сервисы в контейнер
builder.Services.AddHttpContextAccessor();

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services.AddControllers();

var app = builder.Build();
// Настройте путь для статических файлов
app.UseStaticFiles(new StaticFileOptions
{
    FileProvider = new PhysicalFileProvider(
        Path.Combine(builder.Environment.WebRootPath, "uploads")),
    RequestPath = "/uploads"
}); 

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

### Задание 5: Тестирование API
1. Используйте Postman или Swagger для тестирования:
   - Загрузите файл POST /api/mediafiles/upload
   - Получите список файлов GET /api/mediafiles
   - Скачайте файл GET /api/mediafiles/{id}/download
   - Удалите файл DELETE /api/mediafiles/{id}

## Задание
1. Добавьте возможность поиска файлов по описанию
2. Загрузите работу на GitHub.

# Методы ограничения загружаемых файлов в ASP.NET Core

## 1. Ограничение размера файла

### Глобальное ограничение (в Program.cs)
```csharp
builder.Services.Configure<FormOptions>(options =>
{
    options.MultipartBodyLengthLimit = 50 * 1024 * 1024; // 50MB
    options.ValueLengthLimit = 10 * 1024 * 1024; // 10MB для одного файла
});
```

### Ограничение на уровне контроллера/действия
```csharp
[RequestSizeLimit(10 * 1024 * 1024)] // 10MB
[HttpPost("upload")]
public IActionResult UploadFile(IFormFile file)
```

### Ограничение для конкретного действия
```csharp
[DisableRequestSizeLimit] // Отключает ограничение
[HttpPost("upload-large")]
public IActionResult UploadLargeFile(IFormFile file)
```

## 2. Ограничение по сигнатуре файла (магическим числам)

Более надежный способ проверки типа файла:

```csharp
private bool IsValidFileSignature(IFormFile file)
{
    using var reader = new BinaryReader(file.OpenReadStream());
    var signatures = new Dictionary<string, List<byte[]>>
    {
        { ".jpeg", new List<byte[]> { new byte[] { 0xFF, 0xD8, 0xFF, 0xE0 } } },
        { ".png", new List<byte[]> { new byte[] { 0x89, 0x50, 0x4E, 0x47 } } },
        { ".pdf", new List<byte[]> { new byte[] { 0x25, 0x50, 0x44, 0x46 } } }
    };
    
    var ext = Path.GetExtension(file.FileName).ToLower();
    if (!signatures.ContainsKey(ext)) return false;
    
    var headerBytes = reader.ReadBytes(signatures[ext].Max(m => m.Length));
    return signatures[ext].Any(signature => 
        headerBytes.Take(signature.Length).SequenceEqual(signature));
}
```

## 3. Ограничение количества файлов

```csharp
[HttpPost("upload-multiple")]
public IActionResult UploadFiles(List<IFormFile> files)
{
    if (files.Count > 5) // Не более 5 файлов
    {
        return BadRequest("Максимальное количество файлов - 5");
    }
    // ...
}
```

## 4. Ограничение по разрешению изображений (для изображений)

```csharp
using (var image = Image.Load(file.OpenReadStream()))
{
    if (image.Width > 5000 || image.Height > 5000)
    {
        return BadRequest("Максимальное разрешение изображения 5000x5000");
    }
}
```

## 5. Ограничение по имени файла

```csharp
var invalidChars = Path.GetInvalidFileNameChars();
if (file.FileName.Any(c => invalidChars.Contains(c)))
{
    return BadRequest("Имя файла содержит недопустимые символы");
}
```

## 6. Ограничение частоты загрузки

Для защиты от DDoS:

```csharp
[RequestRateLimit(Name = "UploadLimit", Seconds = 60, Count = 5)]
[HttpPost("upload")]
public IActionResult UploadFile(IFormFile file)
```

## Рекомендации по безопасности

1. Всегда переименовывайте загружаемые файлы (используйте GUID)
2. Храните файлы вне корневой папки веб-приложения
3. Ограничивайте права доступа к загруженным файлам
4. Используйте антивирусное сканирование для загружаемых файлов
5. Для критичных систем рассматривайте sandbox-решения для проверки файлов
6. **Используйте другие серивисы для хранения файлов**
