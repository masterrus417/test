# Django Base Project

**Django Base Project** — шаблон Django-проекта, ориентированный на масштабируемость, модульность и использование внутренних библиотек. Создан для корпоративных и изолированных сред, где недопустимо использование сторонних зависимостей из интернета.

## 📦 Структура проекта

```
django-base-project/
├── .vscode/                  # Настройки редактора (опционально)
├── data/                     # Внешние или временные данные
├── example/                  # Пример Django-приложения
│   ├── migrations/
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── models.py
│   ├── tests.py
│   └── views.py
├── libs/                     # Локальные библиотеки проекта
│   ├── db_backend/           # Кастомный database backend
│   ├── middleware/           # Пользовательские middleware
│   ├── prettylog/            # Модуль логгирования с цветами/иконками
│   ├── utils/                # Вспомогательные утилиты
│   └── context.py
├── main/                     # Основное Django-приложение + настройки
│   ├── __init__.py
│   ├── asgi.py
│   ├── urls.py
│   ├── wsgi.py
│   ├── settings/             # Конфигурация проекта
│   │   ├── __init__.py
│   │   ├── base.py           # Базовые настройки
│   │   ├── router.py         # Логика выбора подключения к БД
│   │   ├── local_dev.py      # Настройки для локальной разработки
│   │   ├── django_private.py # Секретные данные (в .git не добавляется)
│   │   ├── local_dev.example.py
│   │   └── django_private.example.py
├── media/                    # Файлы, загружаемые пользователями
├── static/                   # Статические файлы (JS, CSS и т.д.)
├── db.sqlite3                # SQLite база (по умолчанию)
├── manage.py
├── .gitignore
├── gitlab-ci.yml             # Конфигурация CI/CD для GitLab
└── README.md
```

## 🚀 Возможности

- Настройки разбиты на `base`, `local_dev`, `private` и легко расширяются
- Кастомный backend подключения к базе данных (`libs/db_backend`)
- Расширяемая система логирования с цветами и иконками (`libs/prettylog`)
- Поддержка модульной структуры и изолированных компонентов
- Встроенная интеграция с GitLab CI (`gitlab-ci.yml`)
- Все зависимости и логика — локально, без доступа к внешним PyPI

## 🔧 Быстрый старт

```bash
git clone https://your.git.repo/django-base-project.git
cd django-base-project

cp main/settings/local_dev.example.py main/settings/local_dev.py
cp main/settings/django_private.example.py main/settings/django_private.py

python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

python manage.py migrate
python manage.py runserver
```

## 💡 Подходит для

- Изолированных и импортозамещённых сред (например, Astra Linux)
- Разработки внутренних веб-сервисов
- Быстрого прототипирования с переиспользуемыми компонентами
- Django-проектов с особенными требованиями к архитектуре

