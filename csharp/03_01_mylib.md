# Создаем свою либу
1. Создание базовых функций `Add`, `Subtract`, `Multiply`, `Divide`.
2. Реализуйте возведение в степень `Power` и нахождения квадратного корня `SquareRoot` **не** использую сторонние библиотеки ( ~~Math~~ )
3. Реализуйте 3 перегрузки для этих методов (double, int, string)
4. Функции всегда должны возвращать результат или выбрасывать ошибку.
5. Сделайте summary комментарии с помощью нейронной сети.
6. Настройте конфиг:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <PackageId>MyAwesomeLibrary</PackageId>
    <Version>1.0.0</Version>
    <Authors>YourName</Authors>
    <Company>YourCompany</Company>
    <Description>An awesome library that does amazing things</Description>
    <PackageTags>awesome utility tools</PackageTags>
    <PackageLicenseExpression>MIT</PackageLicenseExpression>
    <RepositoryUrl>https://github.com/yourname/MyAwesomeLibrary</RepositoryUrl>
    <GeneratePackageOnBuild>true</GeneratePackageOnBuild>
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
    <DocumentationFile>bin\$(Configuration)\$(TargetFramework)\$(PackageId).xml</DocumentationFile>
  </PropertyGroup>
</Project>
```
7. Зарегистрируйте аккаунт на NuGet.org -> Получите API Key
8. Соберите пакет ```dotnet pack --configuration Release```
9. Опубликуйте пакет: ```dotnet nuget push bin\Release\*.1.0.0.nupkg --api-key YOUR_API_KEY --source https://api.nuget.org/v3/index.json```
10. Добавьте свою любу в другой консольный проект через пакетный менеджер Nuget.
---
### Дополнительные задачи:
1. Реализуйте метод SuperSum складывающий значения "как есть". Пример (SuperSum(21, 12) -> 2112)
   - метод должен иметь 2 перегрузки (double, int, string) и всегда возвращать double.
3. Реализуйте метод Calculator() принимающий один аргумент строка и возвращаюйщий результат этой строки (double). Например "2 + 2" возвращает 4. "10 + 2 * 4" возвращает 48;
4. Оформите README.md на NeGet.org (документацию xml > сконвертуруйте в md)
5. Выгрузите это как патч 1.1.0
