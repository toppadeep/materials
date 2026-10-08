# Как веб-приложение общается с API: Axios и HTTP-запросы

Веб-приложение обычно не хранит все данные внутри себя. Оно получает данные с сервера через **API**.

Упрощённо это выглядит так:

```text
Vue-приложение
      ↓
   HTTP-запрос
      ↓
      API
      ↓
   Сервер / БД
      ↓
   HTTP-ответ
      ↓
Vue-приложение
```

Например, пользователь открыл страницу товаров.

Vue отправляет запрос:

```text
GET /api/products
```

Сервер отвечает:

```json
[
  {
    "id": 1,
    "name": "iPhone",
    "price": 900
  },
  {
    "id": 2,
    "name": "MacBook",
    "price": 1200
  }
]
```

Vue получает этот JSON и выводит его на страницу.

---

# 1. Что такое HTTP-запрос

Запрос от браузера к серверу состоит из нескольких основных частей:

```text
HTTP-запрос
│
├── Метод
├── URL
├── Headers
├── Parameters
└── Body
```

Например:

```text
POST https://example.com/api/products?page=1

Headers:
Authorization: Bearer token123
Content-Type: application/json

Body:
{
  "name": "iPhone",
  "price": 900
}
```

То есть браузер говорит серверу:

> «Я хочу выполнить определённое действие, вот адрес, вот дополнительные данные и вот информация о том, кто я и в каком формате отправляю данные».

---

# 2. HTTP-методы

Основные методы, которые нужно знать:

```text
GET     → получить данные
POST    → создать данные
PUT     → полностью изменить данные
PATCH   → частично изменить данные
DELETE  → удалить данные
```

Например:

```text
GET    /api/products
POST   /api/products
GET    /api/products/10
PATCH  /api/products/10
DELETE /api/products/10
```

Можно представить это так:

```text
GET     → дай мне товар
POST    → создай товар
PUT     → замени товар
PATCH   → измени товар
DELETE  → удали товар
```

---

# 3. Что такое Axios

Axios — это библиотека для выполнения HTTP-запросов из JavaScript/TypeScript.

Установка:

```bash
npm install axios
```

После этого:

```ts
import axios from "axios";
```

Самый простой GET-запрос:

```ts
const response = await axios.get("https://example.com/api/products");
```

Ответ сервера находится в:

```ts
response.data;
```

Например:

```ts
const response = await axios.get<Product[]>("/api/products");

console.log(response.data);
```

---

# 4. GET-запрос

GET используется, когда нужно **получить данные**.

```ts
const response = await axios.get("/api/products");
```

Полученные данные:

```ts
const products = response.data;
```

Можно сразу деструктурировать:

```ts
const { data } = await axios.get<Product[]>("/api/products");
```

Теперь:

```ts
data;
```

содержит массив товаров.

---

# 5. Query-параметры

Иногда нужно передать серверу дополнительные параметры.

Например:

```text
/api/products?page=2&limit=10
```

В Axios это обычно передаётся через `params`:

```ts
const response = await axios.get("/api/products", {
  params: {
    page: 2,
    limit: 10,
  },
});
```

Axios сам сформирует:

```text
/api/products?page=2&limit=10
```

Ещё пример:

```ts
const response = await axios.get("/api/products", {
  params: {
    category: "phone",
    minPrice: 500,
    maxPrice: 1000,
  },
});
```

Получится примерно:

```text
/api/products?category=phone&minPrice=500&maxPrice=1000
```

---

# 6. Query-параметры и Path-параметры

Это важно не путать.

### Query

```text
/api/products?id=10
```

Параметр находится после `?`.

В Axios:

```ts
axios.get("/api/products", {
  params: {
    id: 10,
  },
});
```

### Path

```text
/api/products/10
```

Здесь `10` является частью URL:

```ts
const id = 10;

axios.get(`/api/products/${id}`);
```

Часто API выглядит так:

```text
GET /api/products       → все товары
GET /api/products/10    → конкретный товар
```

---

# 7. POST-запрос

POST обычно используется для создания нового объекта.

Например:

```ts
const response = await axios.post("/api/products", {
  name: "iPhone",
  price: 900,
  category: "phone",
});
```

Здесь второй аргумент — это **body запроса**.

То есть мы отправляем серверу:

```json
{
  "name": "iPhone",
  "price": 900,
  "category": "phone"
}
```

---

# 8. Body запроса

Body — это данные, которые мы отправляем серверу.

Например:

```ts
axios.post("/api/users", {
  name: "Alex",
  email: "alex@example.com",
  age: 25,
});
```

Сервер получает JSON:

```json
{
  "name": "Alex",
  "email": "alex@example.com",
  "age": 25
}
```

То есть:

```text
params → обычно параметры URL

body → данные самого запроса
```

---

# 9. PUT и PATCH

Допустим, у нас есть товар:

```text
/api/products/10
```

### PUT

Обычно используется для полной замены объекта:

```ts
await axios.put("/api/products/10", {
  name: "iPhone 15",
  price: 900,
  category: "phone",
  stock: 10,
});
```

### PATCH

Обычно используется для частичного изменения:

```ts
await axios.patch("/api/products/10", {
  price: 850,
});
```

В данном случае мы говорим:

> Измени только цену товара.

---

# 10. DELETE

Удаление:

```ts
await axios.delete("/api/products/10");
```

Здесь `10` — ID товара.

---

# 11. Headers

Headers — это дополнительная информация о запросе.

Например:

```ts
axios.get("/api/products", {
  headers: {
    Authorization: "Bearer token123",
  },
});
```

Сервер получает:

```text
Authorization: Bearer token123
```

Headers часто используются для:

- авторизации;
- указания формата данных;
- передачи токенов;
- различных служебных параметров.

---

# 12. Content-Type

Один из важных headers:

```text
Content-Type
```

Он сообщает серверу, в каком формате отправляются данные.

Например:

```text
Content-Type: application/json
```

означает:

> Body запроса содержит JSON.

При работе с Axios JSON обычно обрабатывается автоматически.

Например:

```ts
axios.post("/api/users", {
  name: "Alex",
});
```

Axios отправит данные в подходящем формате.

---

# 13. Authorization

Очень часто API требует авторизацию.

Например, сервер выдал пользователю:

```text
token123
```

Тогда запрос может выглядеть так:

```ts
axios.get("/api/profile", {
  headers: {
    Authorization: `Bearer ${token}`,
  },
});
```

То есть:

```text
Authorization: Bearer token123
```

Сервер использует этот токен, чтобы понять, кто делает запрос.

---

# 14. Ответ сервера

Сервер возвращает не только данные.

Axios получает объект `response`:

```ts
const response = await axios.get("/api/products");
```

В нём есть, например:

```ts
response.data;
response.status;
response.headers;
```

Самое важное:

```ts
response.data;
```

Это данные, которые вернул API.

Например:

```ts
const response = await axios.get<Product[]>("/api/products");

console.log(response.data);
```

---

# 15. HTTP status

Сервер сообщает результат запроса через HTTP status.

Основные:

```text
200 → успешно
201 → создано
204 → успешно, но без данных

400 → неправильный запрос
401 → не авторизован
403 → доступ запрещён
404 → не найдено

500 → ошибка сервера
```

Например:

```text
GET /api/products/100
```

Если такого товара нет:

```text
404 Not Found
```

---

# 16. Обработка ошибок

При работе с Axios запрос может завершиться ошибкой.

Поэтому часто используется:

```ts
try {
  const response = await axios.get("/api/products");

  console.log(response.data);
} catch (error) {
  console.log("Ошибка запроса");
}
```

В реальном приложении можно показать пользователю:

```text
Не удалось загрузить товары
```

---

# 17. Axios + Vue

Например, нужно загрузить товары при открытии компонента:

```ts
import { onMounted, ref } from "vue";
import axios from "axios";

const products = ref<Product[]>([]);

const loadProducts = async () => {
  try {
    const response = await axios.get<Product[]>("/api/products");

    products.value = response.data;
  } catch (error) {
    console.error(error);
  }
};

onMounted(() => {
  loadProducts();
});
```

Логика здесь простая:

```text
компонент открылся
       ↓
loadProducts()
       ↓
axios.get()
       ↓
API
       ↓
response.data
       ↓
products.value
       ↓
Vue обновляет интерфейс
```

---

# 18. Axios instance

В реальном проекте не очень удобно каждый раз писать полный URL и headers.

Поэтому создают отдельный Axios instance:

```ts
import axios from "axios";

const api = axios.create({
  baseURL: "https://example.com/api",
});
```

Теперь:

```ts
api.get("/products");
```

вместо:

```ts
axios.get("https://example.com/api/products");
```

---

# 19. Общие headers

Например, можно создать API-клиент с общими настройками:

```ts
const api = axios.create({
  baseURL: "https://example.com/api",
  headers: {
    "Content-Type": "application/json",
  },
});
```

Теперь эти настройки используются для запросов через `api`.

---

# 20. Полный пример CRUD

Допустим, API предоставляет:

```text
GET    /products
GET    /products/:id
POST   /products
PATCH  /products/:id
DELETE /products/:id
```

Тогда клиент может выглядеть так:

```ts
const getProducts = () => api.get("/products");

const getProduct = (id: number) => api.get(`/products/${id}`);

const createProduct = (product: Product) => api.post("/products", product);

const updateProduct = (id: number, data: Partial<Product>) =>
  api.patch(`/products/${id}`, data);

const deleteProduct = (id: number) => api.delete(`/products/${id}`);
```

Теперь Vue-компонент может использовать эти функции.

---

# 21. Самая важная схема

Нужно понимать разницу между четырьмя вещами:

```text
URL
↓
КУДА отправляем запрос

METHOD
↓
ЧТО хотим сделать

PARAMS
↓
дополнительные параметры URL

BODY
↓
какие данные отправляем

HEADERS
↓
дополнительная информация о запросе
```

Например:

```ts
axios.post(
  "/api/products",
  {
    name: "iPhone",
    price: 900,
  },
  {
    headers: {
      Authorization: "Bearer token123",
    },
  },
);
```

Здесь:

```text
POST
        → метод

/api/products
        → URL

{name, price}
        → body

Authorization
        → header
```

А для GET:

```ts
axios.get("/api/products", {
  params: {
    page: 2,
    limit: 10,
  },
  headers: {
    Authorization: "Bearer token123",
  },
});
```

Здесь:

```text
/api/products
        → URL

page=2&limit=10
        → params

Authorization
        → headers
```

---

# Главное, что нужно запомнить

Когда фронтенд общается с API, он постоянно делает примерно одно и то же:

```text
1. Формируем запрос
        ↓
2. Указываем HTTP-метод
        ↓
3. Указываем URL
        ↓
4. При необходимости передаём params
        ↓
5. При необходимости передаём body
        ↓
6. При необходимости передаём headers
        ↓
7. Получаем response
        ↓
8. Берём response.data
        ↓
9. Сохраняем данные в Vue state
        ↓
10. Vue обновляет интерфейс
```

В Axios основные методы:

```ts
axios.get();
axios.post();
axios.put();
axios.patch();
axios.delete();
```

А основные части запроса:

```ts
axios.get(url, {
  params: {},
  headers: {},
});

axios.post(url, data, {
  params: {},
  headers: {},
});
```

То есть **Axios — это просто инструмент, который позволяет JavaScript/Vue удобно отправлять HTTP-запросы и получать ответы от API**.
