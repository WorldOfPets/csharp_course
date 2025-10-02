## Практическое занятие №6: Скрипты развёртывания и CI/CD с GitHub Actions

### Цель занятия
- Написать скрипт для полного развёртывания проекта «с нуля» на Windows (`.bat`)  
- Настроить GitHub Actions, который при пуше в любую ветку:  
  - устанавливает зависимости  
  - применяет миграции  
  - запускает тесты  
- Обеспечить, чтобы мерж в `master` был невозможен при падении тестов

---

### Продолжительность  
~90 минут

---

### Предварительные требования
- Завершено занятие №5  
- Проект содержит модели, шаблоны, представления и тесты  
- Репозиторий на GitHub с включёнными GitHub Actions  
- Локальная ветка `master` актуальна

---

## Часть 1: Создание скрипта развёртывания для Windows (.bat)

1. В корне проекта создайте файл `setup.bat` со следующим содержимым:
```bat
@echo off
echo Установка зависимостей и запуск миграций...

REM Создаём виртуальное окружение, если его нет
if not exist venv (
    python -m venv venv
)

REM Активируем виртуальное окружение и устанавливаем зависимости
call venv\Scripts\activate
pip install -r req.txt

REM Применяем миграции
python manage.py migrate

REM Запускаем тесты
python manage.py test

echo Готово!
pause
```

2. Проверьте работу скрипта:
- Удалите папку `venv` и файл `db.sqlite3` (если есть)  
- Запустите `setup.bat` двойным кликом или из терминала  
- Убедитесь, что тесты проходят

> ⚠️ Убедитесь, что Python доступен в PATH вашей системы.

---

## Часть 2: Настройка GitHub Actions

1. Создайте папку и файл для workflow:
```bash
mkdir -p .github/workflows
```

2. Создайте файл `.github/workflows/test.yml`:
```yaml
name: Django CI

on:
  push:
    branches: [ "**" ]
  pull_request:
    branches: [ master ]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      max-parallel: 4
      matrix:
        python-version: [3.12]

    steps:
    - uses: actions/checkout@v4

    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v5
      with:
        python-version: ${{ matrix.python-version }}

    - name: Install dependencies
      run: |
        python -m venv venv
        source venv/bin/activate
        pip install -r req.txt

    - name: Run migrations
      run: |
        source venv/bin/activate
        python manage.py migrate

    - name: Run tests
      run: |
        source venv/bin/activate
        python manage.py test
```

3. Сохраните файл и добавьте его в Git:
```bash
git add .github/workflows/test.yml setup.sh setup.bat
git commit -m "ci: add setup scripts and GitHub Actions workflow"
```

4. Отправьте изменения в новую ветку:
```bash
git checkout -b ci/setup-and-tests
git push -u link ci/setup-and-tests
```

5. Создайте Pull Request в `master`  
GitHub автоматически запустит workflow. Дождитесь зелёного статуса.

6. Убедитесь, что в настройках репозитория включена **защита ветки `master`**:  
   1. Перейдите в ваш репозиторий на GitHub.
  2. Нажмите **Settings** → **Branches** (в левом меню).
  3. В разделе **Branch protection rules** нажмите **Add rule**.
  4. В поле **Branch name pattern** введите:  
     ```
     master
     ```
  5. Включите следующие опции:
  
     **Require a pull request before merging**  
     → Поставьте галочку.  
     → (Опционально) Укажите, сколько апрувов нужно (например, 1).  
     → **Dismiss stale pull request approvals when new commits are pushed** — хорошая практика.
  
     **Require status checks to pass before merging**  
     → Нажмите **Search for status checks…** и выберите вашу проверку:  
     ```
     build (3.12)
     ```
     (Если не видите — сначала запустите PR, чтобы GitHub "увидел" чек.)
  
     ✅ **Require branches to be up to date before merging**  
     → Рекомендуется, чтобы избежать конфликтов после merge.

---

## Задания для самостоятельной работы

1. **Проверьте отказоустойчивость**:  
   - Временно добавьте в один из тестов `assertFalse`  
   - Запушьте изменения в ветку  
   - Убедитесь, что GitHub Actions **падает**, а PR в `master` **заблокирован**

2. **Улучшите скрипт setup.bat**:  
   - Eсли `req.txt` отсутствует — вывести ошибку и завершить работу  
   - Добавьте клонирование проекта из репозитория

3. После исправления теста (удаления `assertFalse`) — завершите PR и слейте в `master`.

---

## Контрольный список (чек-лист)

- [ ] Файл `setup.bat` создан и работает на Windows  
- [ ] Файл `setup.sh` создан, исполняемый и работает на Unix  
- [ ] Оба скрипта устанавливают зависимости, применяют миграции и запускают тесты  
- [ ] Файл `.github/workflows/test.yml` настроен корректно  
- [ ] GitHub Actions запускается при пуше и в PR  
- [ ] Тесты проходят в CI  
- [ ] Защита ветки `master` настроена: мерж запрещён при падении тестов  
- [ ] Все файлы добавлены в репозиторий и закоммичены  
- [ ] Pull Request создан, проверен и смёржен
