## Практическое занятие №3: Модели Django, миграции и базовое тестирование моделей

### Цель занятия
- Научиться создавать модели данных в Django  
- Применять миграции для обновления базы данных  
- Писать простые unit-тесты для моделей  
- Корректно работать с изменениями в Git (включая миграции)

---

### Продолжительность  
~90 минут

---

### Предварительные требования
- Завершено занятие №2  
- Рабочий Django-проект с приложением `posts`  
- Активное виртуальное окружение  
- Репозиторий на GitHub, ветка `master` актуальна

---

## Часть 1: Создание модели Post

1. Откройте файл `posts/models.py` и определите модель:
```python
from django.db import models
from django.utils import timezone

class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    created_at = models.DateTimeField(default=timezone.now)

    def __str__(self):
        return self.title
```

2. Создайте миграцию:
```bash
python manage.py makemigrations
```
В папке `posts/migrations/` появится файл вроде `0001_initial.py`.

3. Примените миграцию:
```bash
python manage.py migrate
```

4. Зарегистрируйте модель в админке  
Файл `posts/admin.py`:
```python
from django.contrib import admin
from .models import Post

admin.site.register(Post)
```

5. Создайте суперпользователя и проверьте админку:
```bash
python manage.py createsuperuser
python manage.py runserver
```
Зайдите в http://127.0.0.1:8000/admin, добавьте один пост вручную.

---

## Часть 2: Тестирование модели Post

1. Откройте файл `posts/tests.py` и замените его содержимое:
```python
from django.test import TestCase
from .models import Post
from django.utils import timezone

class PostModelTest(TestCase):

    def setUp(self):
        self.post = Post.objects.create(
            title="Тестовый заголовок",
            content="Тестовое содержимое"
        )

    def test_post_creation(self):
        """Проверка, что пост создаётся корректно"""
        self.assertEqual(self.post.title, "Тестовый заголовок")
        self.assertEqual(self.post.content, "Тестовое содержимое")
        self.assertTrue(self.post.created_at <= timezone.now())

    def test_post_str(self):
        """Проверка метода __str__"""
        self.assertEqual(str(self.post), "Тестовый заголовок")
```

2. Запустите тесты:
```bash
python manage.py test
```
Вы должны увидеть, что все тесты прошли успешно.

---

## Часть 3: Работа с Git — коммит изменений с миграциями

1. Создайте новую ветку для модели поста:
```bash
git checkout -b feature/post-model
```

2. Добавьте новые файлы и изменения:
```bash
git add .
```
> Обратите внимание: файл миграции (`0001_initial.py`) **нужно коммитить**, а `db.sqlite3` — **нет** (он в .gitignore).

3. Сделайте коммит:
```bash
git commit -m "feat: add Post model with tests"
```

4. Отправьте ветку на GitHub:
```bash
git push -u link feature/post-model
```

5. Создайте Pull Request на GitHub  
- Base: `master`  
- Compare: `feature/post-model`  
- Описание:  
  ```
  - Добавлена модель Post
  - Реализованы базовые тесты модели
  - Настроена админка
  ```

6. После мержа обновите локальную ветку master:
```bash
git checkout master
git pull link master
```

7. Удалите локальную ветку:
```bash
git branch -d feature/post-model
```

---

## Задания для самостоятельной работы

1. Добавьте поле `author` в модель `Post` (тип `CharField`, max_length=100)  
   - Создайте новую ветку `feature/post-author`  
   - Обновите модель, создайте и примените новую миграцию  
   - Обновите тесты: проверьте, что автор сохраняется  
   - Сделайте коммит, отправьте PR, слейте в master

2. Убедитесь, что при запуске тестов **не используется основная база данных**:  
   Выполните:
   ```bash
   python manage.py test
   ```
   Проверьте, что в выводе есть `Creating test database...` и `Destroying test database...`

3. Проверьте статус репозитория перед коммитом:
   ```bash
   git status
   ```
   Убедитесь, что `db.sqlite3` не отслеживается.

---

## Контрольный список (чек-лист)

- [ ] Модель `Post` создана и содержит поля `title`, `content`, `created_at`  
- [ ] Миграция создана и применена  
- [ ] Модель зарегистрирована в админке  
- [ ] Написаны и проходят тесты для модели  
- [ ] Файл миграции добавлен в коммит  
- [ ] `db.sqlite3` не попал в коммит (проверено через `.gitignore`)  
- [ ] Работа выполнена в отдельной ветке  
- [ ] Pull Request создан и смёржен  
- [ ] Локальная ветка `master` обновлена
