# Лабораторная работа: Реализация чата с использованием SignalR и Redis

## Цель работы
Научиться создавать веб-приложение с реальным временем взаимодействия на основе SignalR, используя Redis для бэкплейна и масштабирования.

## Теоретическая часть

### SignalR в ASP.NET Core
SignalR - это библиотека для добавления функциональности реального времени в веб-приложения. Она использует WebSockets, Server-Sent Events и Long Polling в зависимости от возможностей клиента и сервера.

### Redis Backplane
Redis может использоваться как backplane для SignalR, позволяя масштабировать приложение на несколько серверов, синхронизируя сообщения между ними.

## Практическая часть

### 1. Создание проекта и установка пакетов
- Microsoft.AspNetCore.SignalR
- Microsoft.AspNetCore.SignalR.StackExchangeRedis

### 2. Настройка SignalR и Redis в Program.cs

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowSpecificOrigin",
        builder => builder.WithOrigins("http://127.0.0.1:5500")
                         .AllowCredentials()
                         .AllowAnyHeader()
                         .AllowAnyMethod());
});

// Добавление SignalR
var connectionString = builder.Configuration.GetConnectionString("Redis");
builder.Services.AddSignalR().AddStackExchangeRedis(connectionString, options => {
    options.Configuration.ChannelPrefix = "MyApp_";
    options.Configuration.AbortOnConnectFail = false;
});

builder.Services.AddControllers();
var app = builder.Build();
app.UseCors("AllowSpecificOrigin");
app.MapControllers();
app.MapHub<ChatHub>("/chatHub"); // Добавляем маршрут для хаба

app.Run();
```

### 3. Добавление строки подключения в appsettings.json

```json
{
  "ConnectionStrings": {
    "Redis": "localhost:6379"
  }
}
```

### 4. Создание ChatHub

Создайте `Hubs/ChatHub.cs`:

```csharp
using Microsoft.AspNetCore.SignalR;

public class ChatHub : Hub
{
    private static int _userCount = 0;

    public async Task SendMessage(string user, string message)
    {
        await Clients.All.SendAsync("ReceiveMessage", user, message);
    }

    public override async Task OnConnectedAsync()
    {
        Interlocked.Increment(ref _userCount);
        await Clients.All.SendAsync("UpdateUserCount", _userCount);
        await base.OnConnectedAsync();
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        Interlocked.Decrement(ref _userCount);
        await Clients.All.SendAsync("UpdateUserCount", _userCount);
        await base.OnDisconnectedAsync(exception);
    }

    public Task<int> GetUserCount()
    {
        return Task.FromResult(_userCount);
    }
}
```

### 5. Создание контроллера для управления сообщениями

Создайте `Controllers/ChatController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.SignalR;

[ApiController]
[Route("api/[controller]")]
public class ChatController : ControllerBase
{
    private readonly IHubContext<ChatHub> _hubContext;

    public ChatController(IHubContext<ChatHub> hubContext)
    {
        _hubContext = hubContext;
    }

    [HttpPost("broadcast/{message}")]
    public async Task<IActionResult> BroadcastMessage(string message)
    {
        await _hubContext.Clients.All.SendAsync("ReceiveMessage", "System", message);
        return Ok();
    }

    [HttpGet("usercount")]
    public async Task<IActionResult> GetUserCount()
    {
        //Получаем количество подключенных пользователей
        
    }
}
```

### 6. Клиентская часть (HTML/JavaScript)

Создайте `index.html` в отдельной папке:

```html
<!DOCTYPE html>
<html>
<head>
    <title>SignalR Chat</title>
    <style>
        #messages { height: 300px; overflow-y: auto; border: 1px solid #ccc; padding: 10px; }
        #userCount { font-weight: bold; }
    </style>
</head>
<body>
    <h1>SignalR Chat</h1>
    <div>Online users: <span id="userCount">0</span></div>
    <div id="messages"></div>
    <input type="text" id="userInput" placeholder="Your name">
    <input type="text" id="messageInput" placeholder="Your message">
    <button id="sendButton">Send</button>

    <script src="https://cdnjs.cloudflare.com/ajax/libs/microsoft-signalr/6.0.1/signalr.min.js"></script>
    <script>
        const connection = new signalR.HubConnectionBuilder()
            .withUrl("http://localhost:5143/chatHub")
            .configureLogging(signalR.LogLevel.Information)
            .build();

        connection.on("ReceiveMessage", (user, message) => {
            const msg = document.createElement("div");
            msg.textContent = `${user}: ${message}`;
            document.getElementById("messages").appendChild(msg);
        });

        connection.on("UpdateUserCount", (count) => {
            document.getElementById("userCount").textContent = count;
        });

        connection.start().catch(err => console.error(err.toString()));

        document.getElementById("sendButton").addEventListener("click", async () => {
            const user = document.getElementById("userInput").value;
            const message = document.getElementById("messageInput").value;
            try {
                await connection.invoke("SendMessage", user, message);
                document.getElementById("messageInput").value = "";
            } catch (err) {
                console.error(err.toString());
            }
        });
    </script>
</body>
</html>
```

## Тестирование

1. Запустите приложение
2. Откройте несколько вкладок браузера с `index.html`
3. Отправьте сообщение из одной вкладки - оно должно появиться во всех
4. Реализуйте счетчик пользователей (подсказка: Используйте интерфейс)
5. Отправьте системное сообщение через POST /api/chat/broadcast

## Задания
1. Выгрузите проект на GitHub
2. С помощью комманды MONITOR просмотрите состояние REDIS
