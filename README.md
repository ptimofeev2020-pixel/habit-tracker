# Трекер привычек — Docker, CI/CD и деплой

Бэкенд SPA-приложения для трекинга полезных привычек по книге Джеймса Клира «Атомные привычки».
Привычка формулируется так: «я буду [ДЕЙСТВИЕ] в [ВРЕМЯ] в [МЕСТО]».

## Стек

Python, Django 5.2, Django REST Framework, Simple JWT, PostgreSQL, Celery, Redis, Telegram Bot API,
django-cors-headers, drf-spectacular, Docker, Docker Compose, Gunicorn, Nginx, GitHub Actions.

## Модель «Привычка»

Пользователь, место, время, действие, признак приятной привычки, связанная привычка, периодичность
(в днях, по умолчанию 1), вознаграждение, время на выполнение (в секундах), признак публичности.

## Валидаторы

- нельзя одновременно указать вознаграждение и связанную привычку;
- время на выполнение — не больше 120 секунд;
- связанной может быть только приятная привычка;
- у приятной привычки не может быть вознаграждения или связанной привычки;
- выполнять привычку нужно не реже раза в 7 дней (периодичность от 1 до 7).

## Эндпоинты

| Метод | Адрес | Описание |
|---|---|---|
| POST | `/users/register/` | Регистрация |
| POST | `/users/login/` | Авторизация, пара access/refresh токенов |
| POST | `/users/token/refresh/` | Обновление access-токена |
| GET | `/habits/` | Свои привычки, пагинация по 5 (`limit`/`offset`) |
| POST | `/habits/` | Создание привычки |
| GET, PUT, PATCH, DELETE | `/habits/<id>/` | Просмотр, редактирование, удаление своей привычки |
| GET | `/habits/public/` | Публичные привычки (только чтение) |

Документация: `/api/docs/` (Swagger), `/api/redoc/`, `/api/schema/`.

## Права доступа

Все эндпоинты, кроме регистрации и авторизации, доступны только с JWT-токеном.
Пользователь работает по CRUD только со своими привычками. Публичные привычки других
пользователей видны в списке `/habits/public/`, но изменить или удалить их нельзя.

## Telegram-напоминания

1. Создайте бота у [@BotFather](https://t.me/BotFather) и запишите токен в `TELEGRAM_BOT_TOKEN`.
2. Узнайте свой chat id (например, через [@userinfobot](https://t.me/userinfobot)) и укажите его в поле
   `tg_chat_id` при регистрации.
3. Напишите своему боту `/start`, иначе Telegram не даст ему отправлять вам сообщения.

Celery-beat каждую минуту запускает задачу `habits.tasks.send_habit_reminders`: она находит полезные
привычки, время которых наступило, учитывает периодичность и отправляет напоминание в Telegram.

## Контейнеры

| Сервис | Назначение | Порт |
|---|---|---|
| `nginx` | принимает запросы, отдаёт статику, проксирует в Gunicorn | `80` наружу |
| `web` | Django + Gunicorn; при старте применяет миграции и собирает статику | `8000` только внутри сети |
| `db` | PostgreSQL 16, данные в томе `postgres_data` | `5432` только внутри сети |
| `redis` | брокер Celery, данные в томе `redis_data` | `6379` только внутри сети |
| `celery` | воркер, отправляет напоминания в Telegram | — |
| `celery-beat` | планировщик, раз в минуту ставит задачу напоминаний | — |

`web`, `celery` и `celery-beat` используют один образ `habits-web`. База и Redis проверяются healthcheck'ом,
приложение стартует только после того, как они готовы. У всех сервисов `restart: always`.

## Запуск локально через Docker (одной командой)

Нужен Docker Desktop (Windows/macOS) или Docker Engine с плагином Compose (Linux).

```
cp .env.template .env
docker compose up -d --build
```

В `.env` достаточно задать `SECRET_KEY`, `DB_PASSWORD` и при необходимости `TELEGRAM_BOT_TOKEN`.
Адреса базы и Redis внутри Docker подставляются в `docker-compose.yml` автоматически.

- API: http://localhost/
- Swagger: http://localhost/api/docs/
- Админка: http://localhost/admin/

```
docker compose exec web python manage.py createsuperuser
docker compose ps
docker compose logs -f web
docker compose logs -f celery
docker compose exec web python manage.py test
docker compose down
docker compose down -v
```

Последняя команда удаляет и тома с данными.

## Запуск локально без Docker

```
python -m venv venv
venv\Scripts\activate
pip install -r requirements.txt
cp .env.template .env
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```

Нужны запущенные PostgreSQL и Redis (`DB_HOST=localhost`). В отдельных терминалах:

```
celery -A config worker -l info -P solo
celery -A config beat -l info
```

(`-P solo` нужен на Windows.)

## Тесты и стиль кода

```
coverage run --source='.' --omit='*/migrations/*,manage.py,config/*sgi.py' manage.py test
coverage report
flake8 .
```

Покрытие — 99%, flake8 замечаний не выдаёт.

## CI/CD (GitHub Actions)

Файл `.github/workflows/ci.yml`. Запускается при push в `main`, `develop`, `feature/**` и при pull request
в `main` и `develop`. Этапы идут строго по очереди, следующий стартует только если предыдущий прошёл:

1. **lint** — `flake8 .`.
2. **test** — поднимает PostgreSQL и Redis как сервисы GitHub Actions, ставит зависимости, запускает тесты
   с подсчётом покрытия. Любой упавший тест останавливает pipeline.
3. **build** — собирает Docker-образ, проверяет его (`manage.py check`), проверяет `docker compose config`,
   сохраняет образ как артефакт.
4. **deploy** — только для push: по SSH копирует образ, `docker-compose.yml` и конфиг Nginx на сервер,
   создаёт там `.env` из GitHub Secrets, загружает образ, перезапускает контейнеры и проверяет,
   что `/api/docs/` отвечает.

### Секреты репозитория

Settings → Secrets and variables → Actions → New repository secret:

| Secret | Значение |
|---|---|
| `SSH_HOST` | публичный IP сервера |
| `SSH_USER` | пользователь на сервере |
| `SSH_KEY` | приватный SSH-ключ целиком, вместе со строками BEGIN/END |
| `SECRET_KEY` | секретный ключ Django |
| `DB_PASSWORD` | пароль PostgreSQL |
| `TELEGRAM_BOT_TOKEN` | токен Telegram-бота |

Секреты в репозиторий не попадают: `.env` есть только на сервере и локально, он исключён в `.gitignore`.

## Настройка сервера (Yandex Cloud, Ubuntu 22.04/24.04)

1. Создать SSH-ключ для деплоя на своём компьютере:
   ```
   ssh-keygen -t ed25519 -f ~/.ssh/habits_deploy
   ```
   Passphrase оставить пустой. Публичный ключ `habits_deploy.pub` указать при создании ВМ,
   приватный `habits_deploy` положить в секрет `SSH_KEY`.

2. Подключиться:
   ```
   ssh -i ~/.ssh/habits_deploy USER@SERVER_IP
   ```

3. Установить Docker и Docker Compose, разрешить работу без sudo, включить автозапуск:
   ```
   sudo apt update && sudo apt upgrade -y
   curl -fsSL https://get.docker.com | sudo sh
   sudo usermod -aG docker $USER
   sudo systemctl enable --now docker
   ```
   Переподключиться по SSH и проверить: `docker compose version`.

4. Оставить открытыми только SSH и HTTP:
   ```
   sudo ufw default deny incoming
   sudo ufw default allow outgoing
   sudo ufw allow OpenSSH
   sudo ufw allow 80/tcp
   sudo ufw --force enable
   ```

5. Запретить вход по паролю и под root — в `/etc/ssh/sshd_config`:
   ```
   PasswordAuthentication no
   PermitRootLogin no
   ```
   затем `sudo systemctl restart ssh`.

6. Добавить секреты в GitHub и сделать push — проект развернётся в `~/habits` на сервере.

Проверка и администрирование на сервере:

```
cd ~/habits
docker compose ps
docker compose logs -f web
docker compose exec web python manage.py createsuperuser
```

Приложение доступно по адресу `http://SERVER_IP/`, документация — `http://SERVER_IP/api/docs/`.
После перезагрузки сервера всё поднимается само: Docker включён в systemd, у контейнеров `restart: always`.
