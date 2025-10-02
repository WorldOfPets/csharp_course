## Практическое занятие №5: Шаблоны, статика и работа с удалённым репозиторием

### Цель занятия
- Научиться использовать шаблоны Django вместо «сырого» HTML в представлениях  
- Подключить и использовать статические файлы (CSS)  
- Освоить совместную работу с GitHub: клонирование, пулл, пуш, pull request  
- Обеспечить корректную структуру проекта для командной разработки

---

### Продолжительность  
~90 минут

---

### Предварительные требования
- Завершено занятие №4  
- В проекте есть модели `Post`, представления `home_view`, `post_list_view`, `post_detail_view`  
- Все изменения слиты в ветку `master`  
- Активное виртуальное окружение

---

## Часть 1: Настройка шаблонов

1. Создайте папку для шаблонов:
```bash
mkdir -p posts/templates/posts
```

2. Создайте базовый шаблон `posts/templates/posts/base.html`:
```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>Мой блог</title>
    {% load static %}
    <link rel="stylesheet" type="text/css" href="{% static 'css/style.css' %}">
</head>
<body>
    <header>
        <h1><a href="/">Мой блог</a></h1>
    </header>
    <main>
        {% block content %}
        {% endblock %}
    </main>
</body>
</html>
```

3. Создайте шаблон списка постов `posts/templates/posts/post_list.html`:
```html
{% extends "posts/base.html" %}

{% block content %}
<h2>Все посты</h2>
<ul>
    {% for post in posts %}
        <li>
            <a href="/posts/{{ post.id }}/"><strong>{{ post.title }}</strong></a>:
            {{ post.content|truncatewords:10 }}
        </li>
    {% empty %}
        <li>Постов пока нет.</li>
    {% endfor %}
</ul>
{% endblock %}
```

4. Создайте шаблон детальной страницы `posts/templates/posts/post_detail.html`:
```html
{% extends "posts/base.html" %}

{% block content %}
<h2>{{ post.title }}</h2>
<p>{{ post.content }}</p>
<p><small>Опубликовано: {{ post.created_at|date:"d.m.Y H:i" }}</small></p>
<a href="/posts/">← Назад к списку</a>
{% endblock %}
```

5. Обновите представления в `posts/views.py`, чтобы они использовали шаблоны:
```python
from django.shortcuts import render, get_object_or_404
from .models import Post

def home_view(request):
    return render(request, 'posts/base.html')

def post_list_view(request):
    posts = Post.objects.all()
    return render(request, 'posts/post_list.html', {'posts': posts})

def post_detail_view(request, post_id):
    post = get_object_or_404(Post, id=post_id)
    return render(request, 'posts/post_detail.html', {'post': post})
```

> Обратите внимание: `get_object_or_404` автоматически вернёт 404, если пост не найден.

---

## Часть 2: Настройка статических файлов

1. Создайте папку для статики:
```bash
mkdir -p posts/static/css
```

2. Создайте файл `posts/static/css/style.css`:
```css
body {
    font-family: Arial, sans-serif;
    max-width: 800px;
    margin: 0 auto;
    padding: 20px;
    background-color: #f9f9f9;
}

header h1 a {
    color: #333;
    text-decoration: none;
}

main {
    background: white;
    padding: 20px;
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

ul {
    list-style-type: none;
    padding: 0;
}

li {
    margin-bottom: 15px;
    padding-bottom: 15px;
    border-bottom: 1px solid #eee;
}
```

3. Убедитесь, что в `blog/settings.py` есть настройки для статики (по умолчанию они есть):
```python
STATIC_URL = '/static/'
```

4. Проверьте работу в браузере:
```bash
python manage.py runserver
```
Откройте `/posts/` — страница должна выглядеть стилизованной.

---

## Часть 3: Работа с удалённым репозиторием

1. Переключитесь на `master` и создайте новую ветку:
```bash
git checkout master
git pull link master
git checkout -b feature/templates-and-static
```

2. Добавьте новые файлы:
```bash
git add .
```

3. Сделайте коммит:
```bash
git commit -m "feat: implement templates and static CSS"
```

4. Отправьте ветку на GitHub:
```bash
git push -u link feature/templates-and-static
```

5. Создайте Pull Request на GitHub  
- Base: `master`  
- Compare: `feature/templates-and-static`  
- Описание:  
  ```
  - Добавлены шаблоны: base.html, post_list.html, post_detail.html
  - Подключена статика (style.css)
  - Представления обновлены для использования render()
  ```

6. После мержа обновите локальную `master`:
```bash
git checkout master
git pull link master
```

7. Удалите локальную ветку:
```bash
git branch -d feature/templates-and-static
```

---

## Задания для самостоятельной работы

1. Создайте шаблон для главной страницы (`home.html`), который наследуется от `base.html` и содержит краткое приветствие.  
   - Обновите `home_view`, чтобы он рендерил `posts/home.html`  
   - Добавьте ссылку «Все посты» в шапку или основное содержимое

2. Убедитесь, что статические файлы корректно обслуживаются в режиме разработки  
   - Проверьте в DevTools браузера (вкладка Network), что `style.css` загружается со статусом 200

3. Протестируйте, что при клонировании проекта на «чистую» машину всё работает:  
   - Создайте временную папку  
   - Выполните:
     ```bash
     git clone <ваш-репозиторий> test_clone
     cd test_clone
     python -m venv venv
     source venv/bin/activate   # или venv\Scripts\activate
     pip install -r req.txt
     python manage.py migrate
     python manage.py runserver
     ```
   - Убедитесь, что сайт запускается и стили применяются

4. Выполните всё в новой ветке `feature/home-template`, создайте PR, слейте и обновите `master`.

---

## Контрольный список (чек-лист)

- [ ] Создана папка `templates/posts` с тремя шаблонами  
- [ ] Создана папка `static/css` со стилями  
- [ ] Представления используют `render()` и передают контекст  
- [ ] Страницы отображаются с оформлением  
- [ ] Статические файлы подключены через `{% static %}`  
- [ ] `home_view` также использует шаблон  
- [ ] Все изменения сделаны в отдельной ветке  
- [ ] Pull Request создан и смёржен  
- [ ] Локальная ветка `master` обновлена  
- [ ] Проект корректно клонируется и запускается «с нуля»
