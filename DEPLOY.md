# Инструкция по деплою

## Что нужно перед началом

- SSH доступ к серверу (логин, пароль или ключ)
- Код запушен на GitHub (приватный репо — ок)
- GitHub токен с правами `repo` (Settings → Developer settings → Personal access tokens → classic)
- Docker и Docker Compose установлены на сервере

---

## 1. Подключиться к серверу

```bash
ssh toppadeep@139.100.237.135
```

Вводишь пароль — символы не отображаются, это нормально.

---

## 2. Клонировать репо (первый раз)

```bash
git clone https://<токен>@github.com/toppadeep/app.git
cd app
```

Токен вставляется прямо в URL — GitHub увидит его и даст доступ к приватному репо.

---

## 3. Создать .env файл

```bash
nano .env
```

Вставить:

```dotenv
POSTGRES_DB=salary_db
POSTGRES_USER=salary_user
POSTGRES_PASSWORD=salary_pass
DATABASE_URL=postgresql://salary_user:salary_pass@db:5432/salary_db
```

`Ctrl+O` → Enter → `Ctrl+X` — сохранить и выйти.

> ⚠️ `.env` не хранится в git намеренно — его нужно создавать на сервере вручную каждый раз.

---

## 4. Поднять контейнеры

```bash
docker compose up --build -d
```

- `--build` — пересобрать образы из Dockerfile
- `-d` — запустить в фоне (detached), не занимать терминал

Ждёшь пока все 8 контейнеров покажут `✔`. Особенно важно что `app-api-1` стал `Healthy` — это значит миграции прошли и сервер отвечает.

---

## 5. Засидить начальных пользователей (только первый раз)

```bash
docker compose exec api python -m app.seed.init_users
```

Создаст двух пользователей в БД если их ещё нет. Безопасно запускать повторно — дубликаты не создаст.

---

## 6. Проверить что всё работает

```bash
docker compose ps
```

Все контейнеры должны быть `running`. Потом открыть https://sandbox.students.devupschool.ru/

---

## Следующие деплои (обновление кода)

```bash
ssh toppadeep@139.100.237.135
cd app
git pull
docker compose up --build -d
```

Вот и всё. `git pull` подтягивает изменения, `--build` пересобирает только то что изменилось.

> ⚠️ Если менял модели БД — новая миграция применится автоматически через `alembic upgrade head` в entrypoint.sh при старте контейнера.

---

## Полезные команды

```bash
# Посмотреть логи всех контейнеров
docker compose logs -f

# Логи конкретного контейнера
docker compose logs -f api
docker compose logs -f nginx

# Остановить всё
docker compose down

# Остановить и удалить данные БД (осторожно!)
docker compose down -v

# Зайти внутрь контейнера
docker compose exec api sh
docker compose exec db sh

# Статус контейнеров
docker compose ps
```

---

## На что обращать внимание

| Ситуация | Что делать |
|---|---|
| `app-api-1` не становится Healthy | `docker compose logs api` — смотреть ошибки |
| Сайт не открывается | Проверить что порт 81 свободен: `ss -tlnp \| grep 81` |
| Ошибка миграций | `docker compose exec api alembic upgrade head` вручную |
| Нужно обновить код | `git pull && docker compose up --build -d` |
| `.env` пропал после git pull | Создать заново через `nano .env` — он не в git |
