### **1. Банковский аккаунт**  
**Цель:** Инкапсуляция, свойства, методы.  
**Задача:**  
- Создайте класс `BankAccount` с полями: `AccountNumber`, `OwnerName`, `Balance`.  
- Добавьте методы:  
  - `Deposit(decimal amount)` – пополнение счёта.  
  - `Withdraw(decimal amount)` – снятие денег (проверяйте, достаточно ли средств).  
  - `DisplayBalance()` – вывод информации о балансе.  
- Используйте свойства (`get`/`set`) для контроля доступа к полям.  

**Задание:**  
- Добавьте процент на остаток (`InterestRate`).  
- Реализуйте историю операций (список транзакций).  

---

### **2. Система управления животными в зоопарке**  
**Цель:** Наследование, полиморфизм.  
**Задача:**  
- Создайте базовый класс `Animal` с полями: `Name`, `Age`, `Sound`.  
- Добавьте метод `MakeSound()`, который выводит звук животного.  
- Создайте производные классы: `Lion`, `Elephant`, `Monkey` (переопределите `MakeSound()`).  
- Создайте класс `Zoo`, который содержит список животных и методы:  
  - `AddAnimal(Animal animal)`  
  - `ListAnimals()` – вывод всех животных и их звуков.  

**Задание:**  
- Добавьте интерфейс `IFeedable` с методом `Feed()`, который реализуют некоторые животные.
- Добавьте в класс `Zoo` функцию `FeedAnimals()` которая будет кормить всех кормибельных животных.

---

### **3. Геометрические фигуры**  
**Цель:** Абстрактные классы, переопределение методов.  
**Задача:**  
- Создайте абстрактный класс `Shape` с методами:  
  - `abstract double Area()`  
  - `abstract double Perimeter()`  
- Реализуйте классы `Circle`, `Rectangle`, `Triangle` (наследуйтесь от `Shape`).  
- Создайте список фигур и выведите их площади и периметры.  

---

### **4. Библиотека и книги**  
**Цель:** Композиция, инкапсуляция.  
**Задача:**  
- Создайте класс `Book` с полями: `Title`, `Author`, `Year`, `IsAvailable`.  
- Создайте класс `Library`, который содержит список книг (`List<Book>`).  
- Добавьте методы:  
  - `AddBook(Book book)`  
  - `BorrowBook(string title)` – помечает книгу как недоступную.  
  - `ReturnBook(string title)` – возвращает книгу.  
  - `ListAvailableBooks()` – выводит доступные книги.  

**Доп. задание:**  
- Добавьте класс `Reader` с именем и списком взятых книг.  

---

### **5. Игра «Персонажи RPG»**  
**Цель:** Полиморфизм, интерфейсы.  
**Задача:**  
- Создайте базовый класс `Character` с полями: `Name`, `Health`, `Strength`.  
- Добавьте метод `Attack(Character target)`, который уменьшает здоровье цели.  
- Создайте классы `Warrior`, `Mage`, `Archer` с уникальными способностями.  
- Добавьте интерфейс `ISpecialAbility` с методом `UseSpecialAbility()`. Для каждого героя определите свою уникальную способность.
- Переопределите метод `ToString()` для Character. Вызывайте его при создании персонажа, атаке (выводит цель), и при использовании способностей.
- Определите метод `Die` в классе `Character` и вызывайте его при нулевом здоровье.

**Пример:**  
```csharp
Warrior warrior = new Warrior("Conan", 100, 15);
Mage mage = new Mage("Gandalf", 80, 20);
warrior.Attack(mage);
mage.UseSpecialAbility(); // Например, восстанавливает здоровье
```

---

### **6. Система заказов в магазине**  
**Цель:** Агрегация, работа со списками.  
**Задача:**  
- Создайте класс `Product` (`Id`, `Name`, `Price`).  
- Создайте класс `Order` (`OrderId`, `List<Product>`, `CustomerName`).  
- Добавьте методы:  
  - `AddProduct(Product p)`  
  - `GetTotalPrice()` – возвращает сумму заказа.  
- Создайте класс `OrderManager` для управления заказами.  

**Задание:**  
- Добавьте статус заказа (`Processing`, `Shipped`, `Delivered`) через `enum`.

---
# Контрольная
### Этап 1. Начало
1. Реализуйте игру "Угадай число" (программа генерирует рандомное число - пользователь угадывает)
  - Числа от 0 до 10
  - Число попыток 3
  - После неудачной попытки игра пишет "Загаданное число больше/меньше"
  - По истечению 3 попытко - Game over
  - Угадав число пишет - Your super puper mega ultra 
2. Не забывайте про Issue и Pull-request с последующим мерджем.
3. **Ограничения**: нельзя использовать ООП и классы. Код пишем в Main.
### Этап 2. Улучшам игру
1. Пользователь сам устанавливает диапазон одинм числом (начало всегда 0). Например 42 - получается от 0 до 42. Если пользователь не установил число - использовать **10** по умолчанию.
2. Число попыток просчитывает программ в зависимости от размера. Например для диапазона 0-10 нужно минимум 4 попытки чтобы угадать число со 100% вероятностью. Значит количестов попытко должно быть 4-1 = 3 чтобы сохранить небольшую рандомность.
3. Если пользователь вводит одно и тоже число - попытка не сгорает, а программа уведомляет его что он лошара.
4. Не забывайте про Issue и Pull-request с последующим мерджем.
5. **Ограничения**: нельзя использовать ООП и классы. Код пишем в Main.
### Этап 3. Рефактор
1. Оборачиваем все в ООП. Код разбиваем на методы.
2. Покрываем все тестами.
3. Используйте паттерн Builder а также добавьте сложность(enum) Easy(Количество попыток достаточно для 100% победы), Medium(Количестов попыток для 100% победы -1), Hard(-2). Количество попыток не может быть меньше 1го. 
4. Не забывайте про Issue и Pull-request с последующим мерджем.
