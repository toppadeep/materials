# Backend API Standards — FastAPI / Python

> **Версия:** 1.1  
> **Статус:** Обязательно к соблюдению  
> **Применимо к:** все новые и рефакторируемые проекты команды

---

## Содержание

1. [Версионирование API](#1-версионирование-api)
2. [Структура сущности и CRUD](#2-структура-сущности-и-crud)
3. [Базовые поля модели](#3-базовые-поля-модели)
4. [Нейминг эндпоинтов](#4-нейминг-эндпоинтов)
5. [Структура ответа и пагинация](#5-структура-ответа-и-пагинация)
6. [Фильтрация, поиск и сортировка](#6-фильтрация-поиск-и-сортировка)
7. [Связи между сущностями](#7-связи-между-сущностями)
8. [Сводные таблицы (Many-to-Many)](#8-сводные-таблицы-many-to-many)
9. [Агрегация данных: List vs Get](#9-агрегация-данных-list-vs-get)
10. [Обработка ошибок](#10-обработка-ошибок)
11. [OpenAPI и operationId](#11-openapi-и-operationid)
12. [Чек-лист разработчика](#12-чек-лист-разработчика)

---

## 1. Версионирование API

Все эндпоинты обязательно располагаются под префиксом версии:

```
/api/v1/<entity>
```

**Примеры:**

```
GET  /api/v1/user
GET  /api/v1/user/{uid}
POST /api/v1/user
```

Версия прописывается на уровне роутера, не на уровне каждого эндпоинта:

```python
from fastapi import APIRouter

router = APIRouter(prefix="/api/v1/user", tags=["User"])
```

Подключение в `main.py`:

```python
from app.routers import user

app.include_router(user.router)
```

**Зачем:** при необходимости выпустить v2 с breaking changes — v1 остаётся рабочим, фронт мигрирует постепенно. Даже если сейчас нет планов на v2 — закладывать версию нужно с первого дня, иначе потом это болезненный рефакторинг.

---

## 2. Структура сущности и CRUD

Для **каждой сущности** в проекте обязательно описывается полный набор CRUD-эндпоинтов. Это требование не опционально — даже для MVP. Основная причина: каждый проект предполагает наличие административного интерфейса, и без CRUD-покрытия любая сущность становится неуправляемой без прямого доступа к базе данных.

Все CRUD-эндпоинты сущности **обязательно располагаются в одном роутере/модуле** — не разбрасывать по разным файлам. Дополнительные (доменные) методы размещаются **там же**.

### Минимальный набор эндпоинтов для сущности

| Метод         | HTTP   | Путь              | Описание                               |
|---------------|--------|-------------------|----------------------------------------|
| `getUser`     | GET    | `/user/{uid}`     | Получить одну запись (полная агрегация)|
| `getUsers`    | GET    | `/user`           | Получить список (минимум полей)        |
| `createUser`  | POST   | `/user`           | Создать запись                         |
| `patchUser`   | PATCH  | `/user/{uid}`     | Обновить запись по uid                 |
| `deleteUser`  | DELETE | `/user/{uid}`     | Удалить по uid                         |

---

## 3. Базовые поля модели

Каждая основная сущность **обязана** содержать следующие поля. Добавляются всегда, без исключений:

```python
import uuid
from datetime import datetime
from sqlalchemy.orm import DeclarativeBase, mapped_column, Mapped
from sqlalchemy import DateTime, func


class Base(DeclarativeBase):
    pass


class TimestampMixin:
    uid: Mapped[uuid.UUID] = mapped_column(
        primary_key=True,
        default=uuid.uuid4,
        index=True,
    )
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        nullable=False,
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
        nullable=False,
    )


class User(TimestampMixin, Base):
    __tablename__ = "user"

    email: Mapped[str] = mapped_column(unique=True, nullable=False)
    name: Mapped[str] = mapped_column(nullable=False)
```

**Почему `created_at` и `updated_at` обязательны:**
- базовые поля для сортировки (`sort_by=created_at`)
- `updated_at` нужен для отладки, аудита и синхронизации кэша
- фронтенд использует оба поля практически всегда

---

## 4. Нейминг эндпоинтов

### Правило

Формат: **`<глагол><Сущность>`** (camelCase, сущность с большой буквы).

### CRUD — стандартные глаголы

| Операция           | Глагол   | Пример        |
|--------------------|----------|---------------|
| Получить одну      | `get`    | `getUser`     |
| Получить список    | `get`    | `getUsers`    |
| Создать            | `create` | `createUser`  |
| Обновить           | `patch`  | `patchUser`   |
| Удалить            | `delete` | `deleteUser`  |

### Доменные (дополнительные) методы

Та же формула: глагол описывает действие, сущность — объект.

| Пример действия              | Нейминг                |
|------------------------------|------------------------|
| Заблокировать пользователя   | `blockUser`            |
| Разблокировать               | `unblockUser`          |
| Верифицировать               | `verifyUser`           |
| Добавить в избранное         | `favoriteUser`         |
| Убрать из избранного         | `unfavoriteUser`       |
| Массовое удаление            | `bulkDeleteUsers`      |

### Массовые операции (bulk)

Когда нужно удалить или обновить несколько записей за один запрос — фронт передаёт массив uid'ов:

```python
from pydantic import BaseModel
import uuid

class BulkDeleteSchema(BaseModel):
    uids: list[uuid.UUID]


@router.delete("/bulk", operation_id="bulkDeleteUsers")
async def bulk_delete_users(
    body: BulkDeleteSchema,
    session: AsyncSession = Depends(get_session),
):
    await session.execute(
        delete(User).where(User.uid.in_(body.uids))
    )
    await session.commit()
```

Без bulk-эндпоинта фронт вынужден слать N отдельных запросов в цикле при, например, массовом удалении строк в таблице. Это недопустимо.

> **Запрещено:** использовать пути вида `/user/get-list`, `/user/do-delete`. Семантика операции — в названии функции (`operationId`), не в пути.

---

## 5. Структура ответа и пагинация

### Базовая структура ответа для списков

Любой эндпоинт, возвращающий коллекцию, **обязан** отдавать:

```json
{
  "items": [],
  "total": 0
}
```

`total` — общее количество записей в выборке **с учётом фильтров**, но **без учёта `limit`/`offset`**. Это нужно не только для пагинации — часто полезно знать размер выборки без загрузки всех данных.

### Pydantic-схема

```python
from typing import Generic, TypeVar
from pydantic import BaseModel

T = TypeVar("T")


class PaginatedResponse(BaseModel, Generic[T]):
    items: list[T]
    total: int
```

Использование:

```python
@router.get("", response_model=PaginatedResponse[UserListSchema], operation_id="getUsers")
async def get_users(...):
    ...
```

### Параметры пагинации

Все list-эндпоинты принимают:

```python
from fastapi import Query

limit: int = Query(default=20, ge=1, le=200)
offset: int = Query(default=0, ge=0)
```

> **Запрещено:** возвращать неограниченные выборки без `limit`. Даже если сейчас в таблице 10 записей — завтра их будет 100 000.

---

## 6. Фильтрация, поиск и сортировка

### Поиск (обязателен для большинства сущностей)

```python
search: str | None = Query(default=None)
```

**Правила реализации поиска:**

- Поиск **нестрогий**: `ILIKE`, поиск по вхождению подстроки
- Искать **по нескольким полям** одновременно
- Соединять условия через `OR`

```python
from sqlalchemy import or_

if search:
    pattern = f"%{search}%"
    query = query.where(
        or_(
            User.name.ilike(pattern),
            User.email.ilike(pattern),
        )
    )
```

### Сортировка

```python
from enum import StrEnum

class UserSortField(StrEnum):
    uid = "uid"
    created_at = "created_at"
    name = "name"

class SortOrder(StrEnum):
    asc = "asc"
    desc = "desc"
```

```python
sort_by: UserSortField = Query(default=UserSortField.created_at)
sort_order: SortOrder = Query(default=SortOrder.desc)
```

Применение:

```python
sort_column = getattr(User, sort_by.value)
if sort_order == SortOrder.desc:
    query = query.order_by(sort_column.desc())
else:
    query = query.order_by(sort_column.asc())
```

> **По умолчанию:** сортировать по `created_at DESC`. Это ожидаемое поведение для большинства интерфейсов.

### Итоговый набор параметров list-эндпоинта

```python
@router.get("", response_model=PaginatedResponse[UserListSchema], operation_id="getUsers")
async def get_users(
    limit: int = Query(default=20, ge=1, le=200),
    offset: int = Query(default=0, ge=0),
    search: str | None = Query(default=None),
    sort_by: UserSortField = Query(default=UserSortField.created_at),
    sort_order: SortOrder = Query(default=SortOrder.desc),
    session: AsyncSession = Depends(get_session),
):
    ...
```

---

## 7. Связи между сущностями

### One-to-Many

Для `getOne`-эндпоинтов всегда возвращать агрегированные данные через `selectinload` или `joinedload`.

```python
from sqlalchemy.orm import selectinload

query = (
    select(User)
    .options(selectinload(User.posts))
    .where(User.uid == uid)
)
```

### Выбор стратегии загрузки

| Стратегия       | Когда использовать                                            |
|-----------------|---------------------------------------------------------------|
| `selectinload`  | Коллекции (one-to-many, many-to-many) — **предпочтительно**  |
| `joinedload`    | Одиночные связи (many-to-one, one-to-one)                     |
| `lazyload`      | **Запрещено в async-контексте**                               |

> **Никогда** не использовать `lazy="select"` в async SQLAlchemy — это гарантированный `MissingGreenlet` в продакшне и неконтролируемые N+1 запросы.

### Каскадное удаление и защита от "висячих" ссылок

Перед тем как реализовать удаление сущности, необходимо явно принять решение: **что происходит со связанными данными?**

Есть два допустимых варианта:

**1. Каскадное удаление** — связанные записи удаляются автоматически:

```python
class User(TimestampMixin, Base):
    __tablename__ = "user"

    posts: Mapped[list["Post"]] = relationship(
        "Post",
        back_populates="user",
        cascade="all, delete-orphan",  # удалить посты вместе с пользователем
    )
```

**2. Запрет удаления при наличии связей** — вернуть ошибку, если на запись ссылаются:

```python
from sqlalchemy.exc import IntegrityError

@router.delete("/{uid}", operation_id="deleteUser")
async def delete_user(uid: uuid.UUID, session: AsyncSession = Depends(get_session)):
    user = await session.get(User, uid)
    if not user:
        raise AppException(status_code=404, code="USER_NOT_FOUND")
    try:
        await session.delete(user)
        await session.commit()
    except IntegrityError:
        await session.rollback()
        raise AppException(status_code=409, code="CANNOT_DELETE_REFERENCED")
```

> **Запрещено** оставлять это на волю базы данных без обработки: необработанный `IntegrityError` вернёт фронтенду 500 вместо понятного сообщения об ошибке. Всегда обрабатывать явно.

---

## 8. Сводные таблицы (Many-to-Many)

### Принципы

Сводная таблица — **вспомогательная структура** для хранения связей. Не является самостоятельной бизнес-сущностью и не должна раздуваться лишними полями.

**Минимальная структура:**

```python
from sqlalchemy import UniqueConstraint, ForeignKey

class UserRole(Base):
    """Роли пользователя."""
    __tablename__ = "user_role"

    uid: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    user_uid: Mapped[uuid.UUID] = mapped_column(ForeignKey("user.uid", ondelete="CASCADE"), nullable=False)
    role_uid: Mapped[uuid.UUID] = mapped_column(ForeignKey("role.uid", ondelete="CASCADE"), nullable=False)

    __table_args__ = (
        UniqueConstraint("user_uid", "role_uid", name="uq_user_role"),
    )
```

**`UniqueConstraint` обязателен** — без него при дублирующем запросе создаётся вторая запись, что ломает логику и данные.

**`ondelete="CASCADE"` обязателен** — при удалении пользователя или роли связи удаляются автоматически.

**Правило для фронтенда:** при удалении связи (например, убрать роль у пользователя) фронт передаёт **только `uid` записи сводной таблицы**:

```
DELETE /api/v1/user-role/{uid}
```

Никаких двойных идентификаторов в теле — это упрощает интерфейс и исключает ошибки.

### Проблема N+1 при Many-to-Many

**Антипаттерн — N+1 в цикле:**

```python
# ❌ Каждый пользователь порождает отдельный запрос к БД за ролями
users = await session.execute(select(User))
for user in users.scalars():
    user.roles  # отдельный SELECT на каждого пользователя
```

**Правильно — один запрос через `selectinload`:**

```python
# ✅ Два запроса суммарно: один за пользователями, один за всеми ролями
query = (
    select(User)
    .options(selectinload(User.roles))
    .limit(limit)
    .offset(offset)
)
result = await session.execute(query)
users = result.scalars().all()
```

> **Правило:** агрегировать данные **декларативно через ORM**, никогда не через циклы с отдельными запросами.

---

## 9. Агрегация данных: List vs Get

### List-эндпоинты (`getUsers`)

- Возвращают **минимальный набор полей** — только то, что нужно для строки в таблице/списке
- **Без агрегации** связанных сущностей
- Цель: высокая производительность при больших выборках

```python
class UserListSchema(BaseModel):
    uid: uuid.UUID
    name: str
    email: str
    created_at: datetime

    model_config = ConfigDict(from_attributes=True)
```

### Get-эндпоинты (`getUser`)

- Возвращают **полный набор данных**: все поля + все связанные сущности
- Агрегация обязательна
- Используется при открытии детальной страницы / редактировании

```python
class UserDetailSchema(BaseModel):
    uid: uuid.UUID
    name: str
    email: str
    created_at: datetime
    updated_at: datetime
    roles: list[RoleSchema]
    posts: list[PostShortSchema]

    model_config = ConfigDict(from_attributes=True)
```

> **Мнемоника:** List — "строка в таблице", Get — "карточка с деталями".

---

## 10. Обработка ошибок

### Единый формат ошибки

Все ошибки возвращаются в одном формате:

```json
{
  "detail": "ERROR_CODE"
}
```

`detail` — строковый код ошибки в `UPPER_SNAKE_CASE`. Фронтенд маппит код на текст на нужном языке через единую таблицу. Это позволяет использовать одну функцию маппинга во всех проектах, а не писать её заново каждый раз.

### Базовый класс исключения

```python
from fastapi import HTTPException


class AppException(HTTPException):
    def __init__(self, status_code: int, code: str):
        super().__init__(status_code=status_code, detail=code)
```

Использование:

```python
raise AppException(status_code=404, code="USER_NOT_FOUND")
```

### Предопределённые исключения

Вместо того чтобы каждый раз писать статус-код и код вручную — использовать готовые классы:

```python
class NotFoundException(AppException):
    def __init__(self, code: str = "NOT_FOUND"):
        super().__init__(status_code=404, code=code)

class ConflictException(AppException):
    def __init__(self, code: str = "CONFLICT"):
        super().__init__(status_code=409, code=code)

class BadRequestException(AppException):
    def __init__(self, code: str = "BAD_REQUEST"):
        super().__init__(status_code=400, code=code)
```

Использование:

```python
raise NotFoundException(code="USER_NOT_FOUND")
raise ConflictException(code="DUPLICATE_RELATION")
```

### Таблица кодов ошибок

Это исчерпывающий список. Каждый разработчик обязан использовать коды **из этого списка**. Добавление нового кода согласовывается с командой.

#### 400 Bad Request

| Код                    | Когда использовать                                             |
|------------------------|----------------------------------------------------------------|
| `BAD_REQUEST`          | Общая ошибка некорректного запроса                             |
| `INVALID_INPUT`        | Некорректное значение поля (вне стандартной валидации Pydantic)|
| `INVALID_UUID`         | Передан невалидный UUID                                        |
| `INVALID_DATE_RANGE`   | Дата начала позже даты окончания                               |

#### 401 Unauthorized

| Код                    | Когда использовать                          |
|------------------------|---------------------------------------------|
| `UNAUTHORIZED`         | Запрос без авторизации                      |
| `TOKEN_EXPIRED`        | Токен истёк                                 |
| `TOKEN_INVALID`        | Токен невалиден                             |

#### 403 Forbidden

| Код                    | Когда использовать                                       |
|------------------------|----------------------------------------------------------|
| `FORBIDDEN`            | Нет прав на действие                                     |
| `ACCESS_DENIED`        | Попытка получить доступ к чужому ресурсу                 |

#### 404 Not Found

| Код                    | Когда использовать                                    |
|------------------------|-------------------------------------------------------|
| `NOT_FOUND`            | Универсальный — когда нет специфичного кода           |
| `USER_NOT_FOUND`       | Пользователь не найден                                |
| `ROLE_NOT_FOUND`       | Роль не найдена                                       |
| `POST_NOT_FOUND`       | Запись (пост, элемент) не найдена                     |

> Шаблон для новых сущностей: `<ENTITY>_NOT_FOUND`

#### 409 Conflict

| Код                         | Когда использовать                                               |
|-----------------------------|------------------------------------------------------------------|
| `ALREADY_EXISTS`            | Запись с такими данными уже существует (уникальный ключ)         |
| `EMAIL_ALREADY_EXISTS`      | Email уже занят                                                  |
| `DUPLICATE_RELATION`        | Связь в сводной таблице уже существует (UniqueConstraint)        |
| `CANNOT_DELETE_REFERENCED`  | Запись нельзя удалить — на неё ссылаются другие записи           |

#### 500 Internal Server Error

| Код                    | Когда использовать                                     |
|------------------------|--------------------------------------------------------|
| `INTERNAL_ERROR`       | Необработанная ошибка сервера                          |

Регистрируется глобальный обработчик:

```python
from fastapi import Request
from fastapi.responses import JSONResponse

@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    # логировать exc в системе мониторинга
    return JSONResponse(status_code=500, content={"detail": "INTERNAL_ERROR"})
```

### Обработка дубля в сводной таблице

```python
from sqlalchemy.exc import IntegrityError

@router.post("", operation_id="createUserRole")
async def create_user_role(
    body: UserRoleCreateSchema,
    session: AsyncSession = Depends(get_session),
):
    relation = UserRole(user_uid=body.user_uid, role_uid=body.role_uid)
    session.add(relation)
    try:
        await session.commit()
    except IntegrityError:
        await session.rollback()
        raise ConflictException(code="DUPLICATE_RELATION")
```

### Пример таблицы маппинга на фронтенде

Один файл на весь проект — и он переиспользуется между проектами:

```typescript
// errors.ts
export const ERROR_MESSAGES: Record<string, string> = {
  // 400
  BAD_REQUEST: "Некорректный запрос",
  INVALID_INPUT: "Некорректное значение поля",
  INVALID_UUID: "Неверный формат идентификатора",
  INVALID_DATE_RANGE: "Дата начала не может быть позже даты окончания",

  // 401
  UNAUTHORIZED: "Необходима авторизация",
  TOKEN_EXPIRED: "Сессия истекла, войдите снова",
  TOKEN_INVALID: "Недействительный токен",

  // 403
  FORBIDDEN: "Недостаточно прав",
  ACCESS_DENIED: "Нет доступа к этому ресурсу",

  // 404
  NOT_FOUND: "Запись не найдена",
  USER_NOT_FOUND: "Пользователь не найден",
  ROLE_NOT_FOUND: "Роль не найдена",
  POST_NOT_FOUND: "Запись не найдена",

  // 409
  ALREADY_EXISTS: "Такая запись уже существует",
  EMAIL_ALREADY_EXISTS: "Этот email уже используется",
  DUPLICATE_RELATION: "Связь уже существует",
  CANNOT_DELETE_REFERENCED: "Невозможно удалить: на запись ссылаются другие данные",

  // 500
  INTERNAL_ERROR: "Внутренняя ошибка сервера",

  // fallback
  UNKNOWN_ERROR: "Неизвестная ошибка",
};

export function getErrorMessage(code?: string): string {
  if (!code) return ERROR_MESSAGES.UNKNOWN_ERROR;
  return ERROR_MESSAGES[code] ?? ERROR_MESSAGES.UNKNOWN_ERROR;
}
```

---

## 11. OpenAPI и operationId

### Проблема

FastAPI по умолчанию генерирует `operationId` на основе имени функции и пути:

```
get_users_api_v1_user_get
```

Автогенератор типов на фронтенде превращает это в:

```typescript
// Тип
GetUsersApiV1UserGetErrors

// Функция
getUsersApiV1UserGet()
```

Это нечитаемо и создаёт хрупкие зависимости от структуры URL.

### Решение — централизованный генератор

Настраивается **один раз** при создании приложения:

```python
from fastapi import FastAPI
from fastapi.routing import APIRoute


def generate_unique_id(route: APIRoute) -> str:
    """
    operationId = имя Python-функции.
    Функции именуются по правилу <глагол><Сущность>,
    поэтому operationId автоматически получает правильный формат.
    """
    return route.name


app = FastAPI(
    title="Project API",
    version="1.0.0",
    generate_unique_id_function=generate_unique_id,
)
```

Результат на фронтенде:

```typescript
// Тип
GetUsersErrors

// Функция
getUsers()
```

### Требования к именованию функций

Раз имя Python-функции становится `operationId` — оно **обязано** быть уникальным в рамках всего приложения:

```python
@router.get("", operation_id="getUsers")
async def get_users(...): ...

@router.get("/{uid}", operation_id="getUser")
async def get_user(...): ...

@router.post("", operation_id="createUser")
async def create_user(...): ...

@router.patch("/{uid}", operation_id="patchUser")
async def patch_user(...): ...

@router.delete("/{uid}", operation_id="deleteUser")
async def delete_user(...): ...
```

> **Это не опционально.** Фронтенд строит автогенерируемые типы и сторы на основе `operationId`. Несоблюдение ломает весь процесс автогенерации.

---

## 12. Чек-лист разработчика

Перед тем как считать эндпоинты для сущности готовыми — пройтись по списку:

### Версионирование и структура

- [ ] Роутер использует префикс `/api/v1/<entity>`
- [ ] CRUD сущности в одном файле/модуле

### Модель

- [ ] Присутствуют поля `uid`, `created_at`, `updated_at`
- [ ] Сводные таблицы имеют `UniqueConstraint` и `ondelete="CASCADE"`

### CRUD

- [ ] Описаны все 5 базовых эндпоинтов (`getUser`, `getUsers`, `createUser`, `patchUser`, `deleteUser`)
- [ ] Доменные методы — в том же роутере
- [ ] Реализован bulk-метод там, где нужна массовая операция

### List-эндпоинт

- [ ] Принимает `limit`, `offset`
- [ ] Принимает `search` (нестрогий, по нескольким полям)
- [ ] Принимает `sort_by`, `sort_order`
- [ ] Возвращает `{ items: [], total: 0 }`
- [ ] Схема ответа — минимальный набор полей (без агрегации)

### Get-эндпоинт

- [ ] Возвращает полную схему с агрегацией
- [ ] Связанные сущности загружаются через `selectinload`/`joinedload`
- [ ] Нет `lazyload` в async-контексте

### Связи и удаление

- [ ] Принято явное решение: каскад или запрет при удалении
- [ ] `IntegrityError` при удалении обрабатывается и возвращает `CANNOT_DELETE_REFERENCED`
- [ ] Дубль в сводной таблице обрабатывается и возвращает `DUPLICATE_RELATION`

### Ошибки

- [ ] Все ошибки используют `AppException` с кодом из утверждённой таблицы
- [ ] Зарегистрирован глобальный обработчик `Exception` → `INTERNAL_ERROR`
- [ ] Нет "голых" `raise HTTPException(...)` — только через `AppException`

### OpenAPI

- [ ] Настроен `generate_unique_id_function` на уровне приложения
- [ ] Все эндпоинты имеют явный `operation_id`
- [ ] Формат: `<глагол><Сущность>` (camelCase)
- [ ] `operationId` уникальны в рамках всего приложения

---

*Вопросы по стандарту и предложения по дополнениям — в общий чат команды.*
