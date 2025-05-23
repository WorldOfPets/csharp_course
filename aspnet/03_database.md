# **Лабораторная работа: Основы работы с базами данных в MS SQL Server Management Studio**  
**Тема:** Проектирование и создание базы данных интернет-магазина  

## **Цель работы**  
Создание реляционной базы данных в **MS SQL Server Management Studio (SSMS)**.  

---

## **Теоретическая часть**  
### **Основные понятия**  
1. **Таблица (Table)** – структура для хранения данных в виде строк (записей) и столбцов (полей).  
2. **Первичный ключ (Primary Key, PK)** – уникальный идентификатор записи в таблице.  
3. **Внешний ключ (Foreign Key, FK)** – поле, которое ссылается на первичный ключ другой таблицы, обеспечивая связь между таблицами.  
4. **Ограничения (Constraints)** – правила, накладываемые на данные (NOT NULL, UNIQUE, CHECK, DEFAULT).  
5. **Индекс (Index)** – структура для ускорения поиска данных.  
6. **Представление (View)** – виртуальная таблица, основанная на результате SQL-запроса.  

---

## **Практическая часть**  
### **Задание 1. Создание базы данных и таблиц**  
1. **Создайте новую базу данных `OnlineStore`.**  
2. **Создайте таблицы со следующими полями:**  

#### **1. Таблица `User`**  
- **UserID** (уникальный идентификатор, автоинкремент, первичный ключ)  
- **Username** (строка, уникальное, не может быть NULL)  
- **Email** (строка, уникальное, проверка на корректность email)  
- **PasswordHash** (хэш пароля, не может быть NULL)  
- **RegistrationDate** (дата и время, значение по умолчанию — текущая дата)  
- **LastLogin** (дата и время, может быть NULL)  

#### **2. Таблица `Permissions`**  
- **PermissionID** (уникальный идентификатор, первичный ключ)  
- **Name** (название права, например, "Admin", "Moderator", "User")  
- **Description** (описание, может быть NULL)  

#### **3. Таблица `ContentType`**  
- **ContentTypeID** (уникальный идентификатор, первичный ключ)  
- **TypeName** (название типа контента, например, "Product", "BlogPost")  

#### **4. Таблица `Group`**  
- **GroupID** (уникальный идентификатор, первичный ключ)  
- **Name** (название группы, не может быть NULL)  
- **Description** (описание, может быть NULL)  

#### **5. Таблица `AdminLog`**  
- **LogID** (уникальный идентификатор, первичный ключ)  
- **UserID** (внешний ключ на таблицу `User`)  
- **Action** (действие, например, "Delete User", "Edit Product")  
- **Timestamp** (дата и время, значение по умолчанию — текущая дата)  
- **Details** (дополнительная информация, может быть NULL)  

#### **6. Таблица `Products`**  
- **ProductID** (уникальный идентификатор, первичный ключ)  
- **Name** (название товара, не может быть NULL)  
- **Description** (описание, может быть NULL)  
- **Price** (цена, не может быть NULL, должна быть положительной)  
- **StockQuantity** (количество на складе, по умолчанию 0) 

#### **7. Таблица `Cart`**  
- **CartID** (уникальный идентификатор, первичный ключ)  
- **UserID** (внешний ключ на таблицу `User`)  
- **CreatedAt** (дата создания, значение по умолчанию — текущая дата)  

#### **8. Таблица `Session`**  
- **SessionID** (уникальный идентификатор, первичный ключ)  
- **UserID** (внешний ключ на таблицу `User`)  
- **Token** (уникальный токен сессии, не может быть NULL)  
- **ExpiresAt** (дата истечения сессии, не может быть NULL)  

#### **9. Таблица `Migrations`**  
- **MigrationID** (уникальный идентификатор, первичный ключ)  
- **Name** (название миграции, не может быть NULL)  
- **AppliedAt** (дата применения, значение по умолчанию — текущая дата)  

#### **10. Таблица `Order`**  
- **OrderID** (уникальный идентификатор, первичный ключ)  
- **UserID** (внешний ключ на таблицу `User`)  
- **OrderDate** (дата заказа, значение по умолчанию — текущая дата)  
- **Status** (статус заказа, например, "Pending", "Completed", "Cancelled")  
- **TotalAmount** (общая сумма заказа, должна быть положительной)  

---

### **Задание 2. Установка связей между таблицами**  
1. **Связь `User` → `Permissions`** (многие ко многим через промежуточную таблицу, если необходимо).  
2. **Связь `User` → `Cart`** (один пользователь может иметь одну корзину).  
3. **Связь `User` → `Order`** (один пользователь может иметь несколько заказов).  
4. **Связь `Products` → `Cart`** (многие ко многим через промежуточную таблицу `CartItems`, если требуется).
5. Установите другие связи по своему усмотрению.

---

### **Задание 3. Создание индексов и представлений**  
1. **Создайте индексы** для ускорения поиска по полям:  
   - `Email` в таблице `User`  
   - `Name` в таблице `Products`  
   - `Status` в таблице `Order`  

2. **Создайте представления:**  
   - `ActiveUsers` – список пользователей, которые заходили за последние 30 дней.  
   - `ProductStock` – товары с количеством на складе менее 10.  
   - `OrderSummary` – сводка по заказам (ID заказа, пользователь, сумма, статус).  
---

## **Доп. задание**  
1. **Создайте триггер**, который автоматически записывает в `AdminLog` при добавлении нового товара.   
 
