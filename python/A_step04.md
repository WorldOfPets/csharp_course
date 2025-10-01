## Практическое занятие №4: Представления (views) и тестирование ответов

### Цель занятия
- Научиться создавать представления (views) для отображения данных из модели  
- Настроить маршрутизацию для списка постов  
- Писать тесты на HTTP-ответы с использованием Django-клиента  
- Применять Git-ветвление при добавлении новой функциональности

---

### Продолжительность  
~90 минут

---

### Предварительные требования
- Завершено занятие №3  
- В проекте есть модель `Post` и базовые тесты  
- Актуальная ветка `master` с рабочим проектом  
- Виртуальное окружение активировано

---

## Часть 1: Создание представления для списка постов

1. Обновите файл `posts/views.py`:
```python
from django.shortcuts import render
from django.http import HttpResponse
from .models import Post

def home_view(request):
    return HttpResponse("<h1>Добро пожаловать в мой блог!</h1>")

def post_list_view(request):
    posts = Post.objects.all()
    html = "<h1>Все посты</h1><ul>"
    for post in posts:
        html += f"<li><strong>{post.title}</strong>: {post.content[:50]}...</li>"
    html += "</ul>"
    return HttpResponse(html)
```

2. Обновите маршрутизацию в `posts/urls.py`:
```python
from django.urls import path
from . import views

urlpatterns = [
    path('', views.home_view, name='home'),
    path('posts/', views.post_list_view, name='post_list'),
]
```

3. Проверьте работу в браузере  
Запустите сервер:
```bash
python manage.py runserver
```
Откройте:  
- http://127.0.0.1:8000/ → главная  
- http://127.0.0.1:8000/posts/ → список постов

Если у вас есть посты (например, через админку), они должны отобразиться.

---

## Часть 2: Тестирование представлений

1. Обновите файл `posts/tests.py`, добавив тесты для `post_list_view`:
```python
from django.test import TestCase, Client
from .models import Post

class PostModelTest(TestCase):
    # ... оставьте существующие тесты без изменений ...

class PostViewTest(TestCase):

    def setUp(self):
        self.client = Client()
        Post.objects.create(title="Тест 1", content="Содержимое 1")
        Post.objects.create(title="Тест 2", content="Содержимое 2")

    def test_post_list_view_status(self):
        """Проверка, что страница /posts/ возвращает статус 200"""
        response = self.client.get('/posts/')
        self.assertEqual(response.status_code, 200)

    def test_post_list_view_content(self):
        """Проверка, что на странице отображаются заголовки постов"""
        response = self.client.get('/posts/')
        self.assertContains(response, "Тест 1")
        self.assertContains(response, "Тест 2")
```

2. Запустите тесты:
```bash
python manage.py test
```
Все тесты должны пройти успешно.

---

## Часть 3: Работа с Git — добавление функциональности в ветке

1. Переключитесь на актуальную master и создайте новую ветку:
```bash
git checkout master
git pull link master
git checkout -b feature/post-list-view
```

2. Добавьте изменения в индекс:
```bash
git add .
```

3. Сделайте коммит с осмысленным сообщением:
```bash
git commit -m "feat: add post list view with tests"
```

4. Отправьте ветку на GitHub:
```bash
git push -u link feature/post-list-view
```

5. Создайте Pull Request на GitHub  
- Base: `master`  
- Compare: `feature/post-list-view`  
- Описание:  
  ```
  - Реализовано представление post_list_view
  - Добавлен маршрут /posts/
  - Написаны тесты на статус и содержимое ответа
  ```

6. После мержа обновите локальную master:
```bash
git checkout master
git pull link master
```

7. Удалите локальную ветку:
```bash
git branch -d feature/post-list-view
```

---

## Задания для самостоятельной работы

1. Создайте представление `post_detail_view`, которое отображает один пост по его `id`  
   - Добавьте маршрут: `path('posts/<int:post_id>/', ...)`  
   - В представлении получите пост через `Post.objects.get(id=post_id)`  
   - Обработайте случай, если пост не найден (пока можно не обрабатывать — допустимо падение 500)  
   - Напишите тест (happy path): проверка статуса 200 и наличия заголовка поста
   - Напишите тест: поведение при запросе несуществующего поста
   - Выполните всё в новой ветке `feature/post-detail-view`

2. Проверьте, что в выводе тестов отображается количество запущенных тестов:
```bash
python manage.py test
```
Убедитесь, что их 4 (2 для модели + 2 для представлений).

---

## Контрольный список (чек-лист)

- [ ] Представление `post_list_view` создано  
- [ ] Маршрут `/posts/` настроен  
- [ ] Страница отображает список постов  
- [ ] Написаны и проходят тесты на статус и содержимое  
- [ ] Использован `Client` из `django.test`  
- [ ] Работа выполнена в отдельной ветке  
- [ ] Коммит содержит только релевантные изменения  
- [ ] Pull Request создан и смёржен  
- [ ] Локальная ветка `master` обновлена
