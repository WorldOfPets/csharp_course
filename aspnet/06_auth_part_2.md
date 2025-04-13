# Лабораторная работа: Добавление функционала постов в API с аутентификацией

## Расширение существующего API для работы с постами

### 1. Добавление модели Post

Создайте новый класс модели в папке `Models`:

```csharp
// Models/Post.cs
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace AuthLab.Models
{
    public class Post
    {
        [Key]
        public int Id { get; set; }
        
        [Required]
        [StringLength(200)]
        public string Title { get; set; }
        
        [Required]
        public string Content { get; set; }
        
        public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
        public DateTime? UpdatedAt { get; set; }
        
        [Required]
        public string AuthorId { get; set; }
        
        [ForeignKey("AuthorId")]
        public IdentityUser Author { get; set; }
        
        public bool IsPublished { get; set; } = false;
    }
}
```

### 2. Обновление контекста базы данных

Модифицируйте `ApplicationDbContext.cs`:

```csharp
using AuthLab.Models;
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;

namespace AuthLab.Data
{
    public class ApplicationDbContext : IdentityDbContext
    {
        public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
            : base(options)
        {
        }
        
        public DbSet<Post> Posts { get; set; }

        protected override void OnModelCreating(ModelBuilder builder)
        {
            base.OnModelCreating(builder);
            
            builder.Entity<Post>()
                .HasOne(p => p.Author)
                .WithMany()
                .HasForeignKey(p => p.AuthorId)
                .OnDelete(DeleteBehavior.Restrict);
        }
    }
}
```

### 3. Создание DTO для постов

Добавьте новые модели в папку `Models`:

```csharp
// Models/PostDto.cs
namespace AuthLab.Models
{
    public class PostDto
    {
        public string Title { get; set; }
        public string Content { get; set; }
        public bool IsPublished { get; set; }
    }
}

// Models/PostResponseDto.cs
namespace AuthLab.Models
{
    public class PostResponseDto
    {
        public int Id { get; set; }
        public string Title { get; set; }
        public string Content { get; set; }
        public DateTime CreatedAt { get; set; }
        public string AuthorName { get; set; }
        public bool IsPublished { get; set; }
    }
}
```

### 4. Создание контроллера для работы с постами

Создайте новый контроллер `PostsController.cs`:

```csharp
using AuthLab.Models;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using System.Security.Claims;

namespace AuthLab.Controllers
{
    [Authorize]
    [Route("api/[controller]")]
    [ApiController]
    public class PostsController : ControllerBase
    {
        private readonly ApplicationDbContext _context;
        private readonly UserManager<IdentityUser> _userManager;
        
        public PostsController(
            ApplicationDbContext context,
            UserManager<IdentityUser> userManager)
        {
            _context = context;
            _userManager = userManager;
        }
        
        // GET: api/posts (только опубликованные посты)
        [AllowAnonymous]
        [HttpGet]
        public async Task<ActionResult<IEnumerable<PostResponseDto>>> GetPublishedPosts()
        {
            return await _context.Posts
                .Where(p => p.IsPublished)
                .Include(p => p.Author)
                .Select(p => new PostResponseDto
                {
                    Id = p.Id,
                    Title = p.Title,
                    Content = p.Content,
                    CreatedAt = p.CreatedAt,
                    AuthorName = p.Author.UserName,
                    IsPublished = p.IsPublished
                })
                .ToListAsync();
        }
        
        // GET: api/posts/my (посты текущего пользователя)
        [HttpGet("my")]
        public async Task<ActionResult<IEnumerable<PostResponseDto>>> GetMyPosts()
        {
            var userName = User.FindFirstValue(ClaimTypes.Name);
        
            return await _context.Posts
                .Where(p => p.Author.UserName == userName)
                .Select(p => new PostResponseDto
                {
                    Id = p.Id,
                    Title = p.Title,
                    Content = p.Content,
                    CreatedAt = p.CreatedAt,
                    AuthorName = p.Author.UserName,
                    IsPublished = p.IsPublished
                })
                .ToListAsync();
        }
        
        // GET: api/posts/5
        [AllowAnonymous]
        [HttpGet("{id}")]
        public async Task<ActionResult<PostResponseDto>> GetPost(int id)
        {
            var post = await _context.Posts
                .Include(p => p.Author)
                .FirstOrDefaultAsync(p => p.Id == id);
        
            if (post == null)
            {
                return NotFound();
            }
        
            // Только автор или админ может видеть неопубликованный пост
            var userName = User.FindFirstValue(ClaimTypes.Name);
            if (!post.IsPublished && post.Author.UserName != userName && !User.IsInRole("Admin"))
            {
                return Forbid();
            }
        
            return new PostResponseDto
            {
                Id = post.Id,
                Title = post.Title,
                Content = post.Content,
                CreatedAt = post.CreatedAt,
                AuthorName = post.Author.UserName,
                IsPublished = post.IsPublished
            };
        }
        
        // POST: api/posts
        [HttpPost]
        public async Task<ActionResult<PostResponseDto>> CreatePost(PostDto postDto)
        {
            var userName = User.FindFirstValue(ClaimTypes.Name);
            var user = await _userManager.FindByNameAsync(userName);
        
            var post = new PostModel
            {
                Title = postDto.Title,
                Content = postDto.Content,
                AuthorId = user.Id,
                IsPublished = postDto.IsPublished
            };
        
            _context.Posts.Add(post);
            await _context.SaveChangesAsync();
        
            return CreatedAtAction(nameof(GetPost), new { id = post.Id },
                new PostResponseDto
                {
                    Id = post.Id,
                    Title = post.Title,
                    Content = post.Content,
                    CreatedAt = post.CreatedAt,
                    AuthorName = user.UserName,
                    IsPublished = post.IsPublished
                });
        }
        
        // PUT: api/posts/5
        [HttpPut("{id}")]
        public async Task<IActionResult> UpdatePost(int id, PostDto postDto)
        {
            var post = await _context.Posts.FindAsync(id);
            if (post == null)
            {
                return NotFound();
            }
        
            var userName = User.FindFirstValue(ClaimTypes.Name);
            if (post.Author.UserName != userName && !User.IsInRole("Admin"))
            {
                return Forbid();
            }
        
            post.Title = postDto.Title;
            post.Content = postDto.Content;
            post.IsPublished = postDto.IsPublished;
            post.UpdatedAt = DateTime.UtcNow;
        
            _context.Entry(post).State = EntityState.Modified;
        
            try
            {
                await _context.SaveChangesAsync();
            }
            catch (DbUpdateConcurrencyException)
            {
                if (!PostExists(id))
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
        
        // DELETE: api/posts/5
        [HttpDelete("{id}")]
        public async Task<IActionResult> DeletePost(int id)
        {
            var post = await _context.Posts.FindAsync(id);
            if (post == null)
            {
                return NotFound();
            }
        
            var userName = User.FindFirstValue(ClaimTypes.Name);
            if (post.Author.UserName != userName && !User.IsInRole("Admin"))
            {
                return Forbid();
            }
        
            _context.Posts.Remove(post);
            await _context.SaveChangesAsync();
        
            return NoContent();
        }
        
        private bool PostExists(int id)
        {
            return _context.Posts.Any(e => e.Id == id);
        }
    }
}
```

### 5. Обновление миграций

Выполните следующие команды для обновления базы данных:

```bash
dotnet ef migrations add AddPostsTable
dotnet ef database update
```

### 6. Тестирование функционала постов

1. **Создание поста** (требуется аутентификация):
```http
POST /api/posts
Authorization: Bearer <your_token>
Content-Type: application/json

{
    "title": "Мой первый пост",
    "content": "Содержание моего первого поста",
    "isPublished": true
}
```

2. **Получение всех опубликованных постов** (доступно без аутентификации):
```http
GET /api/posts
```

3. **Получение своих постов** (требуется аутентификация):
```http
GET /api/posts/my
Authorization: Bearer <your_token>
```

4. **Обновление поста** (только автор или админ):
```http
PUT /api/posts/1
Authorization: Bearer <your_token>
Content-Type: application/json

{
    "title": "Обновленный заголовок",
    "content": "Обновленное содержание",
    "isPublished": false
}
```

### 7. Задание
1. Загрузите проект на GitHub
2. Выполните тестирования с помощью Postman
3. Интегрируйте авторизацию в ваш StoreApi
