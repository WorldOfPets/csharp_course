# Лабораторная работа: Консольный файловый менеджер на C#

## Цель работы
Создать консольное приложение, имитирующее базовые команды Unix/Linux (pwd, ls, cat) для работы с файловой системой Windows.

## Базовые команды

### 1. Команда `pwd` (Print Working Directory)
```csharp
public static void PwdCommand()
{
    Console.WriteLine($"Текущая директория: {Directory.GetCurrentDirectory()}");
}
```

### 2. Команда `ls` (List Directory Contents)
```csharp
public static void LsCommand(string[] args)
{
    string path = args.Length > 1 ? args[1] : Directory.GetCurrentDirectory();
    
    try
    {
        Console.WriteLine($"Содержимое директории {path}:");
        Console.WriteLine("Папки:");
        foreach (var dir in Directory.GetDirectories(path))
        {
            Console.WriteLine($"  {Path.GetFileName(dir)}/");
        }
        
        Console.WriteLine("\nФайлы:");
        foreach (var file in Directory.GetFiles(path))
        {
            var fileInfo = new FileInfo(file);
            Console.WriteLine($"  {fileInfo.Name,-30} {fileInfo.Length,10} байт");
        }
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Ошибка: {ex.Message}");
    }
}
```

### 3. Команда `cat` (Concatenate and Display)
```csharp
public static void CatCommand(string[] args)
{
    if (args.Length < 2)
    {
        Console.WriteLine("Не указан файл для просмотра");
        return;
    }

    string filePath = Path.Combine(Directory.GetCurrentDirectory(), args[1]);
    
    try
    {
        Console.WriteLine($"Содержимое файла {filePath}:");
        Console.WriteLine("----------------------------------------");
        using (var reader = new StreamReader(filePath))
        {
            string line;
            while ((line = reader.ReadLine()) != null)
            {
                Console.WriteLine(line);
            }
        }
        Console.WriteLine("----------------------------------------");
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Ошибка: {ex.Message}");
    }
}
```

## Дополнительные задания

### 1. Реализация команды `mkdir` (Make Directory)
```csharp
public static void MkdirCommand(string[] args)
{
    if (args.Length < 2)
    {
        Console.WriteLine("Не указано имя директории");
        return;
    }

    string dirName = args[1];
    string fullPath = Path.Combine(Directory.GetCurrentDirectory(), dirName);
    
    try
    {
        Directory.CreateDirectory(fullPath);
        Console.WriteLine($"Директория создана: {fullPath}");
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Ошибка: {ex.Message}");
    }
}
```

### 2. Реализация команды `rm` (Remove)
```csharp
public static void RmCommand(string[] args)
{
    if (args.Length < 2)
    {
        Console.WriteLine("Не указан файл или директория для удаления");
        return;
    }

    string target = Path.Combine(Directory.GetCurrentDirectory(), args[1]);
    
    try
    {
        if (Directory.Exists(target))
        {
            Directory.Delete(target, true);
            Console.WriteLine($"Директория удалена: {target}");
        }
        else if (File.Exists(target))
        {
            File.Delete(target);
            Console.WriteLine($"Файл удален: {target}");
        }
        else
        {
            Console.WriteLine($"Файл или директория не найдены: {target}");
        }
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Ошибка: {ex.Message}");
    }
}
```

## Задания

1. Реализуйте команду `cp` (копирование файлов)
2. Реализуйте цветной вывод для разных типов файлов
3. Перепиште команду `ls` - вывод содержимого каталога в одну линию.
4. Реализуйте флаги для одной из комманд.
5. Используйте нейронные сети.
6. Все команды (файлы exe) уложите в один каталог и добавьте в системные переменные (если вы работает из под Unix систем - добавьте `my` к каждой из команд).
