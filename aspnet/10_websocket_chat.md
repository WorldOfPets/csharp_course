# Лабораторная работа: ASP.NET Core WebSocket API с использованием Redis

## Цель работы
Научиться создавать веб-приложение с WebSocket API на ASP.NET Core, используя Redis для публикации/подписки (Pub/Sub) и хранения состояния.

## Теоретическая часть

### WebSocket в ASP.NET Core
WebSocket обеспечивает постоянное двустороннее соединение между клиентом и сервером. В ASP.NET Core WebSocket поддерживается через middleware.

### Redis Pub/Sub
Redis поддерживает модель публикации/подписки, позволяя серверам обмениваться сообщениями через каналы. Это полезно для масштабирования WebSocket приложений.

## Практическая часть

### 1. Создание проекта и установка пакетов

- Microsoft.AspNetCore.WebSockets
- StackExchange.Redis

### 2. Настройка Redis в Program.cs

```csharp
using StackExchange.Redis;

var builder = WebApplication.CreateBuilder(args);

// Добавление Redis
builder.Services.AddSingleton<IConnectionMultiplexer>(_ => 
    ConnectionMultiplexer.Connect(builder.Configuration.GetConnectionString("Redis")));

builder.Services.AddControllers();
var app = builder.Build();

app.UseWebSockets();
app.MapControllers();
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

### 4. Создание WebSocketMiddleware

Создайте `WebSocketMiddleware.cs`:

```csharp
using System.Net.WebSockets;
using System.Text;
using StackExchange.Redis;

public class WebSocketMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IConnectionMultiplexer _redis;

    public WebSocketMiddleware(RequestDelegate next, IConnectionMultiplexer redis)
    {
        _next = next;
        _redis = redis;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        if (context.Request.Path == "/ws")
        {
            if (context.WebSockets.IsWebSocketRequest)
            {
                var webSocket = await context.WebSockets.AcceptWebSocketAsync();
                await HandleWebSocket(webSocket);
            }
            else
            {
                context.Response.StatusCode = StatusCodes.Status400BadRequest;
            }
        }
        else
        {
            await _next(context);
        }
    }

    private async Task HandleWebSocket(WebSocket webSocket)
    {
        var redisSubscriber = _redis.GetSubscriber();
        var channel = "messages";
        
        // Подписка на Redis канал
        await redisSubscriber.SubscribeAsync(channel, (_, message) => 
        {
            var messageBytes = Encoding.UTF8.GetBytes(message);
            webSocket.SendAsync(new ArraySegment<byte>(messageBytes), 
                WebSocketMessageType.Text, true, CancellationToken.None);
        });

        var buffer = new byte[1024 * 4];
        try
        {
            var receiveResult = await webSocket.ReceiveAsync(
                new ArraySegment<byte>(buffer), CancellationToken.None);

            while (!receiveResult.CloseStatus.HasValue)
            {
                var message = Encoding.UTF8.GetString(buffer, 0, receiveResult.Count);
                Console.WriteLine($"Received: {message}");

                // Публикация сообщения в Redis
                await redisSubscriber.PublishAsync(channel, message);

                receiveResult = await webSocket.ReceiveAsync(
                    new ArraySegment<byte>(buffer), CancellationToken.None);
            }

            await webSocket.CloseAsync(
                receiveResult.CloseStatus.Value,
                receiveResult.CloseStatusDescription,
                CancellationToken.None);
        }
        finally
        {
            await redisSubscriber.UnsubscribeAsync(channel);
        }
    }
}
```

### 5. Регистрация middleware в Program.cs

```csharp
var app = builder.Build();

app.UseWebSockets();
app.UseMiddleware<WebSocketMiddleware>(); // Добавьте эту строку

app.MapControllers();
app.Run();
```

### 6. Создание контроллера для управления сообщениями

Создайте `MessagesController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using StackExchange.Redis;

[ApiController]
[Route("api/[controller]")]
public class MessagesController : ControllerBase
{
    private readonly IConnectionMultiplexer _redis;

    public MessagesController(IConnectionMultiplexer redis)
    {
        _redis = redis;
    }

    [HttpPost("broadcast/{message}")]
    public async Task<IActionResult> Broadcast(string message)
    {
        var subscriber = _redis.GetSubscriber();
        await subscriber.PublishAsync("messages", message);
        return Ok();
    }

    [HttpGet("count")]
    public async Task<IActionResult> GetClientCount()
    {
        var server = _redis.GetServer(_redis.GetEndPoints().First());
        var count = server.SubscriptionSubscriberCount("messages");
        return Ok(count);
    }
}
```

### 7. Тестирование

1. Запустите приложение
2. Отправьте сообщение через POST /api/messages/broadcast/your_message
3. Проверьте количество подключений через GET /api/messages/count
```html
<!DOCTYPE html>
<html>
<head>
    <title>WebSocket Test</title>
</head>
<body>
    <h1>WebSocket Test Client</h1>
    <input type="text" id="messageInput" placeholder="Enter message">
    <button onclick="sendMessage()">Send</button>
    <div id="messages"></div>

    <script>
        const socket = new WebSocket('ws://localhost:5179/ws');
        const messagesDiv = document.getElementById('messages');

        socket.onopen = function(e) {
            addMessage('Connection established');
        };

        socket.onmessage = function(event) {
            addMessage(`Server: ${event.data}`);
        };

        socket.onclose = function(event) {
            if (event.wasClean) {
                addMessage(`Connection closed cleanly, code=${event.code}, reason=${event.reason}`);
            } else {
                addMessage('Connection died');
            }
        };

        socket.onerror = function(error) {
            addMessage(`Error: ${error.message}`);
        };

        function sendMessage() {
            const message = document.getElementById('messageInput').value;
            socket.send(message);
            addMessage(`You: ${message}`);
            document.getElementById('messageInput').value = '';
        }

        function addMessage(message) {
            const p = document.createElement('p');
            p.textContent = message;
            messagesDiv.appendChild(p);
        }
    </script>
</body>
</html>
```

## Задания 

1. Выгрузите проект на GitHub.


