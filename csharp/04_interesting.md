### **1. Алгоритмы и структуры данных**  
**Задача:** Напишите реализацию **собственного списка (аналог `List<T>`)**, включая методы:  
- `Add`, `Remove`, `IndexOf`, `Contains`  
- Автоматическое расширение при заполнении внутреннего массива.  

**Дополнительно:** Реализуйте поддержку `IEnumerable<T>` для работы с `foreach`.  

---  

### **2. Работа с файлами и строками**  
**Задача:** Напишите программу, которая читает текстовый файл, находит **10 самых часто встречающихся слов** и выводит их в консоль (игнорируя регистр и знаки препинания).  

**Усложнение:** Используйте многопоточность (`Task` или `Parallel.ForEach`) для ускорения обработки больших файлов.  

---  

### **3. Многопоточность и асинхронность**  
**Задача:** Создайте **симулятор банковских транзакций** с использованием `async/await`:  
- Класс `BankAccount` с методами `Deposit`, `Withdraw` и `GetBalance`.  
- Обеспечьте **потокобезопасность** через `lock` или `SemaphoreSlim`.  
- Сымитируйте 1000 параллельных транзакций с разными задержками (`Task.Delay`).  

---  

### **4. LINQ и делегаты**  
**Задача:** Дан массив чисел. Напишите **LINQ-запрос**, который:  
- Группирует числа по четности.  
- Для каждой группы находит минимум, максимум и среднее значение.  
- Сортирует группы по убыванию количества элементов.  

**Пример:**  
```csharp  
int[] numbers = { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };  
```  

---  

### **5. Работа с API (HTTP-запросы)**  
**Задача:** Напишите программу, которая:  
1. Делает запрос к публичному API (например, [Mock API](https://mockapi.io/)).  
2. Реализуйте CRUD. Для данное АПИ. Также сделайте фильтрацию и поиск по записям.

**Библиотеки:** `HttpClient`, `Newtonsoft.Json` (или `System.Text.Json`).  

---  

### **6. Исключения и валидация**  
**Задача:** Создайте метод, который парсит строку в число (аналог `int.Parse`), но:  
- Бросает **кастомное исключение** `InvalidNumberFormatException`, если строка некорректна.  
- Поддерживает обработку чисел в разных культурах (например, `"1.5"` vs `"1,5"`).  

---  

### **7. Оптимизация кода (алгоритмическая задача)**  
**Задача:** Дан массив из 1_000_000 целых чисел. Напишите **самый быстрый** способ найти:  
- Все уникальные элементы.  
- Элементы, встречающиеся больше 3 раз.  

**Ограничение:** Используйте **HashSet** или **Dictionary** для оптимизации.  

---  

### **8. Reflection и атрибуты**  
**Задача:** Создайте **кастомный атрибут** `[AuthorInfo]`, который хранит имя автора и дату создания класса.  
- Напишите метод, который с помощью **Reflection** выводит все классы с этим атрибутом в сборке.  

---  
