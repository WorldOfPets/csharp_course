## Практическое занятие №2: Первое Django-приложение и управление ветками в Git

### Цель занятия
- Создать первый Django-проект и приложение  
- Научиться работать с ветками в Git  
- Освоить базовый workflow: создание ветки → разработка → коммит → pull request → слияние

---

### Продолжительность
~90 минут (можно разделить на 2 части: 45 мин Django + 45 мин Git)

---

### Предварительные требования
- Установлен Python 3.8+
- Установлен Git
- Настроен виртуальный окружение (из занятия 1)
- Аккаунт на GitHub

---

## Часть 1: Создание Django-проекта и приложения

1. Активируйте виртуальное окружение
```bash
source venv/bin/activate   # Linux/macOS
# или
venv\Scripts\activate      # Windows
```

2. Убедитесь, что Django установлен
```bash
pip install django
pip freeze > req.txt
```

3. Создайте проект и приложение
```bash
django-admin startproject blog .
python manage.py startapp posts
```

4. Зарегистрируйте приложение  
В файле blog/settings.py найдите INSTALLED_APPS и добавьте:
```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    # ... остальные ...
    'posts',
]
```

5. Создайте простую главную страницу

Файл posts/views.py:
```python
from django.http import HttpResponse

def home_view(request):
    return HttpResponse("<h1>Добро пожаловать в мой блог!</h1>")
```

Файл posts/urls.py (создайте его!):
```python
from django.urls import path
from . import views

urlpatterns = [
    path('', views.home_view, name='home'),
]
```

Файл blog/urls.py (обновите):
```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('posts.urls')),
]
```

6. Запустите сервер и проверьте
```bash
python manage.py runserver
```
Откройте в браузере: http://127.0.0.1:8000  
Вы должны увидеть: «Добро пожаловать в мой блог!»

Проверка: сервер запускается, страница отображается.

---

## Часть 2: Работа с Git — ветки и pull request

1. Инициализируйте репозиторий (если ещё не сделано)
```bash
git init
```

2. Создайте .gitignore  
Создайте файл .gitignore в корне проекта:
```gitignore
venv/
__pycache__/
*.pyc
db.sqlite3
.env
.DS_Store
*.log
media/
staticfiles/
```

3. Сделайте первый коммит в master
```bash
git add .
git commit -m "feat: initial Django project setup"
```

4. Создайте новую ветку для главной страницы
```bash
git checkout -b feature/homepage
```

5. Внесите изменения (если ещё не сделали) и закоммитьте
```bash
git add .
git commit -m "feat: add homepage view and routing"
```

6. Отправьте ветку на GitHub  
Сначала создайте репозиторий на GitHub (без README!), затем:
```bash
git remote add link https://github.com/ваш-логин/blog-project.git
git push -u link feature/homepage
```

7. Создайте Pull Request (PR)  
- Перейдите на GitHub → ваш репозиторий  
- Нажмите Compare & pull request  
- Укажите:  
  Title: `feat: add homepage view`  
  Description:  
  ```
  - Добавлено приложение posts
  - Реализован простой view для главной страницы
  - Настроена маршрутизация
  ```
- Нажмите Create pull request

(Если вы работаете в одиночку — просто создайте PR и сразу смёржите его)

8. Слейте PR в master и обновите локально  
После мержа на GitHub:
```bash
git checkout master
git pull link master
```

Теперь master содержит вашу новую функцию!

---

## Задания для самостоятельной работы

1. Добавьте вторую страницу /about/ с текстом «Об авторе»  
   - Создайте новую ветку feature/about-page  
   - Реализуйте view и маршрут  
   - Сделайте коммит и PR  
   - После мержа — обновите локальную master

2. Проверьте историю коммитов:
```bash
git log --oneline --graph
```
Убедитесь, что ветки правильно слились.

3. Удалите локальную и удалённую ветку после мержа:
```bash
git branch -d feature/about-page
git push link --delete feature/about-page
```

---

## Контрольный список (чек-лист)

- [ ] Django-проект создан  
- [ ] Приложение posts зарегистрировано  
- [ ] Главная страница отображается  
- [ ] .gitignore настроен  
- [ ] Работа велась в отдельной ветке  
- [ ] Коммиты имеют осмысленные сообщения  
- [ ] Pull Request создан и смёржен  
- [ ] Локальная ветка master обновлена
