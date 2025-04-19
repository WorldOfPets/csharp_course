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

