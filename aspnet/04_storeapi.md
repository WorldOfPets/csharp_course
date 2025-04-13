# Лабораторная работа: Создание простого API на ASP.NET Core с Entity Framework

## Тема: Разработка RESTful API для системы управления пользователями и товарами

**Цель работы:** Научиться создавать RESTful API с использованием ASP.NET Core и Entity Framework Core, реализовать CRUD-операции для работы с базой данных.

### Часть 1: Настройка проекта

1. Создайте новый проект ASP.NET Core Web API:
2. Установите необходимые пакеты:
   - Microsoft.EntityFrameworkCore
   - Microsoft.EntityFrameworkCore.SqlServer
   - Microsoft.EntityFrameworkCore.Design
   - Microsoft.EntityFrameworkCore.Tools

### Часть 2: Создание моделей и контекста базы данных

1. Создайте папку `Models` и добавьте в нее классы моделей:

```csharp
// User.cs
public class User
{
    public int UserID { get; set; }
    
    [Required]
    [StringLength(50)]
    public string Username { get; set; }
    
    [Required]
    [EmailAddress]
    public string Email { get; set; }
    
    [Required]
    public string PasswordHash { get; set; }
    
    public DateTime RegistrationDate { get; set; } = DateTime.UtcNow;
    
    public DateTime? LastLogin { get; set; }
    
    public ICollection<Permission> Permissions { get; set; }
    public User()
    {
        Permissions = new List<Permission>();
    }
    
    public void AddDefaultPermission(Permission permission)
    {
        if (permission != null && !Permissions.Any(p => p.Name == permission.Name))
        {
            Permissions.Add(permission);
        }
    }
    //public ICollection<Cart> Carts { get; set; }
    //public ICollection<Session> Sessions { get; set; }
    //public ICollection<Order> Orders { get; set; }
    //public ICollection<AdminLog> AdminLogs { get; set; }
}

// Permission.cs
public class Permission
{
    public int PermissionID { get; set; }
    
    [Required]
    [StringLength(50)]
    public string Name { get; set; }
    
    public string Description { get; set; }
    
    public ICollection<User> Users { get; set; }
}

// Остальные модели (ContentType, Group, AdminLog, Product, Cart, Session, Migration, Order)
// должны быть созданы аналогично с соответствующими свойствами и связями
```

2. Создайте класс контекста базы данных:

```csharp
// StoreContext.cs
public class StoreContext : DbContext
{
    public StoreContext(DbContextOptions<StoreContext> options) : base(options) { }
    
    public DbSet<User> Users { get; set; }
    public DbSet<Permission> Permissions { get; set; }
    //public DbSet<ContentType> ContentTypes { get; set; }
    //public DbSet<Group> Groups { get; set; }
    //public DbSet<AdminLog> AdminLogs { get; set; }
    //public DbSet<Product> Products { get; set; }
    //public DbSet<Cart> Carts { get; set; }
    //public DbSet<Session> Sessions { get; set; }
    //public DbSet<Order> Orders { get; set; }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<User>()
            .Property(u => u.UserID)
            .UseIdentityColumn();
        // Настройка связей и ограничений
        modelBuilder.Entity<User>()
            .HasIndex(u => u.Username)
            .IsUnique();
            
        modelBuilder.Entity<User>()
            .HasIndex(u => u.Email)
            .IsUnique();
            
        //modelBuilder.Entity<Product>()
          //  .Property(p => p.Price)
            //.HasColumnType("decimal(18,2)");
            
        // Другие настройки модели...
    }
}
```

### Часть 3: Настройка подключения к базе данных

1. В файле `appsettings.json` добавьте строку подключения:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=StoreDb;Trusted_Connection=True;"
  },
  // ...
}
```

2. В `Program.cs` добавьте сервис контекста базы данных:

```csharp
builder.Services.AddDbContext<StoreContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
//Позволяет работать с вложенными JSON объектами
builder.Services.AddControllers()
    .AddJsonOptions(options =>
    {
        options.JsonSerializerOptions.ReferenceHandler = ReferenceHandler.IgnoreCycles;
        options.JsonSerializerOptions.WriteIndented = true;
    });
```

### Часть 4: Создание контроллеров

1. Создайте контроллер для работы с пользователями:

```csharp
// UsersController.cs
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly StoreContext _context;

    public UsersController(StoreContext context)
    {
        _context = context;
    }

    // GET: api/Users
    [HttpGet]
    public async Task<ActionResult<IEnumerable<User>>> GetUsers()
    {
        return await _context.Users.ToListAsync();
    }

    // GET: api/Users/5
    [HttpGet("{id}")]
    public async Task<ActionResult<User>> GetUser(int id)
    {
        var user = await _context.Users.FindAsync(id);

        if (user == null)
        {
            return NotFound();
        }

        return user;
    }

    // POST: api/Users
    [HttpPost]
    public async Task<ActionResult<User>> PostUser(User user)
    {
        _context.Users.Add(user);
        await _context.SaveChangesAsync();

        return CreatedAtAction(nameof(GetUser), new { id = user.UserID }, user);
    }

    // PUT: api/Users/5
    [HttpPut("{id}")]
    public async Task<IActionResult> PutUser(int id, User user)
    {
        if (id != user.UserID)
        {
            return BadRequest();
        }

        _context.Entry(user).State = EntityState.Modified;

        try
        {
            await _context.SaveChangesAsync();
        }
        catch (DbUpdateConcurrencyException)
        {
            if (!UserExists(id))
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

    // DELETE: api/Users/5
    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteUser(int id)
    {
        var user = await _context.Users.FindAsync(id);
        if (user == null)
        {
            return NotFound();
        }

        _context.Users.Remove(user);
        await _context.SaveChangesAsync();

        return NoContent();
    }

    private bool UserExists(int id)
    {
        return _context.Users.Any(e => e.UserID == id);
    }
}
```

2. Аналогичным образом создайте контроллеры для других сущностей (ProductsController, OrdersController и т.д.)

### Часть 5: Применение миграций и заполнение начальными данными

0. Установите dotnet-ef
   ```bash
   dotnet tool install --global dotnet-ef
   ```
1. Создайте миграцию:
   ```bash
   dotnet ef migrations add InitialCreate
   ```

2. Примените миграцию к базе данных:
   ```bash
   dotnet ef database update
   ```

4. Создайте класс для заполнения начальными данными:

```csharp
public static class DbInitializer
{
    public static void Initialize(StoreContext context)
    {
        context.Database.EnsureCreated();

        if (context.Permissions.Any())
        {
            return; // DB has been seeded
        }

        var permissions = new Permission[]
        {
            new Permission{Name="Admin", Description="Full access"},
            new Permission{Name="Moderator", Description="Moderate content"},
            new Permission{Name="User", Description="Regular user"}
        };

        foreach (var p in permissions)
        {
            context.Permissions.Add(p);
        }
        context.SaveChanges();

        // Добавьте другие начальные данные по аналогии
    }
}
```

4. Вызовите инициализацию в `Program.cs`:

```csharp
using (var scope = app.Services.CreateScope())
{
    var services = scope.ServiceProvider;
    var context = services.GetRequiredService<StoreContext>();
    DbInitializer.Initialize(context);
}
```

### Часть 6: Тестирование API

1. Запустите приложение
2. Используйте Postman или Swagger UI для тестирования API:
   - GET /api/Users - получить список пользователей
   - POST /api/Users - создать нового пользователя
   - GET /api/Users/1 - получить пользователя с ID=1
   - PUT /api/Users/1 - обновить пользователя с ID=1
   - DELETE /api/Users/1 - удалить пользователя с ID=1

### Основное задание

1. Улучшаем метод `GetUsers()` (в данный момент он возвращает объект без разрешений, a также возвращает PasswordHash что не безопасно)
   ```csharp
   // GET: api/Users
    [HttpGet]
    public async Task<ActionResult<IEnumerable<object>>> GetUsers()
    {
        return await _context.Users
            .Include(u => u.Permissions)
            .Select(u => new
            {
                u.UserID,
                u.Username,
                u.Email,
                Permissions = u.Permissions.Select(p => new
                {
                    p.PermissionID,
                    p.Name
                })
            })
            .ToListAsync();
    }
   ```
2. Улучшите подобным образом другие методы Get.
3. Добавьте логирование действий в AdminLog
4. Выгрузите проект на GitHub. Старайтесь отдельно комитить функцию или класс.
5. Протестируйте каждый запрос через `Postman`
   - Создайте отдельную коллекцию
   - Создайте Flow
   - Напишите 5 тестов (GET, GET, POST, PUT, DELETE) для одного из контроллера.
   ```javascript
   pm.test("Status code is 200", function(){
    pm.response.to.have.status(200)
    })
    // 2. Verify response headers
    pm.test("Content-Type is application/json; charset=utf-8", function() {
        pm.expect(pm.response.headers.get("Content-Type")).to.eql("application/json; charset=utf-8");
    });
    
    // 3. Parse response JSON
    const response = pm.response.json();
    
    // 4. Validate user object structure
    pm.test("Response has correct user structure", function() {
        pm.expect(response).to.be.an("object");
        pm.expect(response).to.have.all.keys(
            "userID", 
            "username", 
            "email", 
            "passwordHash", 
            "registrationDate", 
            "lastLogin", 
            "permissions"
        );
    });
   ``` 
