# Работа с JSON-данными в Vue + TypeScript

В реальных проектах данные часто приходят с сервера в формате JSON. Обычно JSON превращается в массив объектов, с которым мы дальше работаем через JavaScript/TypeScript.

Например:

```ts
const products = ref<Product[]>([
  {
    id: 1,
    name: "iPhone 15",
    price: 900,
    category: "phone",
    stock: 10,
  },
  {
    id: 2,
    name: "MacBook Air",
    price: 1200,
    category: "laptop",
    stock: 5,
  },
  {
    id: 3,
    name: "iPad",
    price: 700,
    category: "tablet",
    stock: 0,
  },
]);
```

## 1. Получить все объекты

Просто обращаемся к массиву:

```ts
products.value;
```

В шаблоне Vue:

```vue
<div v-for="product in products" :key="product.id">
  {{ product.name }}
</div>
```

---

## 2. Получить конкретный объект — `find()`

Если нужно найти **один** объект:

```ts
const product = products.value.find((product) => product.id === 2);
```

Результат:

```ts
{
  id: 2,
  name: 'MacBook Air',
  price: 1200,
  ...
}
```

`find()` возвращает первый найденный объект или `undefined`.

---

## 3. Найти несколько объектов — `filter()`

Если нужно получить **массив подходящих объектов**:

```ts
const phones = products.value.filter((product) => product.category === "phone");
```

Результат:

```ts
[
  {
    id: 1,
    name: 'iPhone 15',
    ...
  }
]
```

Главное различие:

```text
find()   → один объект
filter() → массив объектов
```

---

## 4. Фильтрация по нескольким условиям

Условия можно объединять:

```ts
const products = allProducts.value.filter((product) => {
  return (
    product.category === "phone" && product.price < 1000 && product.stock > 0
  );
});
```

Здесь одновременно проверяется:

- категория — `phone`;
- цена меньше `1000`;
- товар есть на складе.

---

## 5. Поиск по тексту

Например, поиск товара по названию:

```ts
const search = "iphone";

const result = products.value.filter((product) =>
  product.name.toLowerCase().includes(search.toLowerCase()),
);
```

`includes()` проверяет, содержит ли строка нужный текст.

---

## 6. Изменить данные с помощью `map()`

`map()` используется, когда нужно пройти по всем объектам и получить **новый массив**.

Например, добавить каждому товару цену с налогом:

```ts
const result = products.value.map((product) => ({
  ...product,
  priceWithTax: product.price * 1.2,
}));
```

Важно:

```ts
{
  ...product
}
```

создаёт новый объект, не изменяя старый.

---

## 7. Изменить конкретный объект

Например, увеличить количество товара:

```ts
const product = products.value.find((product) => product.id === 2);

if (product) {
  product.stock += 1;
}
```

Либо через `map()`:

```ts
products.value = products.value.map((product) =>
  product.id === 2 ? { ...product, stock: product.stock + 1 } : product,
);
```

Второй вариант особенно полезен, когда хочется создать новый массив без изменения старого.

---

## 8. Удалить объект

Например, удалить товар с `id = 2`:

```ts
products.value = products.value.filter((product) => product.id !== 2);
```

Логика простая:

> Оставить все товары, кроме товара с `id = 2`.

---

## 9. Добавить объект

Для добавления используется `push()`:

```ts
products.value.push({
  id: 4,
  name: "AirPods",
  price: 200,
  category: "accessory",
  stock: 15,
});
```

Теперь массив содержит новый объект.

---

## 10. Сортировка — `sort()`

По цене от меньшей к большей:

```ts
products.value.sort((a, b) => a.price - b.price);
```

От большей к меньшей:

```ts
products.value.sort((a, b) => b.price - a.price);
```

По названию:

```ts
products.value.sort((a, b) => a.name.localeCompare(b.name));
```

### Важный момент

`sort()` **изменяет исходный массив**.

Поэтому часто делают копию:

```ts
const sorted = [...products.value].sort((a, b) => a.price - b.price);
```

Это особенно важно при работе с `computed()`.

---

## 11. `computed()` для фильтрации

В Vue фильтр обычно делают через `computed()`:

```ts
const search = ref("");

const filteredProducts = computed(() => {
  return products.value.filter((product) =>
    product.name.toLowerCase().includes(search.value.toLowerCase()),
  );
});
```

В шаблоне:

```vue
<div v-for="product in filteredProducts" :key="product.id">
  {{ product.name }}
</div>
```

Когда `search` изменится, `filteredProducts` автоматически пересчитается.

---

## 12. Фильтрация + сортировка

Можно сначала отфильтровать данные, а потом отсортировать:

```ts
const result = computed(() => {
  return products.value
    .filter((product) => product.stock > 0)
    .sort((a, b) => a.price - b.price);
});
```

Но здесь есть проблема: `sort()` изменит массив, который вернул `filter()` — это не изменит `products.value`, потому что `filter()` уже создал новый массив. Тем не менее для более сложной логики можно явно сделать копию:

```ts
const result = computed(() => {
  return [...products.value]
    .filter((product) => product.stock > 0)
    .sort((a, b) => a.price - b.price);
});
```

---

## 13. `findIndex()`

Иногда нужен не сам объект, а его индекс:

```ts
const index = products.value.findIndex((product) => product.id === 2);
```

Например:

```ts
if (index !== -1) {
  products.value[index].stock += 1;
}
```

Разница:

```text
find()      → возвращает объект
findIndex() → возвращает индекс
```

---

## 14. `some()`

Проверяет, существует ли хотя бы один подходящий элемент:

```ts
const hasAvailableProducts = products.value.some(
  (product) => product.stock > 0,
);
```

Результат:

```ts
true;
```

или

```ts
false;
```

---

## 15. `every()`

Проверяет, подходят ли **все** элементы:

```ts
const allAvailable = products.value.every((product) => product.stock > 0);
```

---

## 16. `reduce()`

Используется, когда из массива нужно получить **одно итоговое значение**.

Например, посчитать общую стоимость товаров:

```ts
const total = products.value.reduce((sum, product) => sum + product.price, 0);
```

Или среднюю цену:

```ts
const average =
  products.value.reduce((sum, product) => sum + product.price, 0) /
  products.value.length;
```

---

# Главное, что нужно запомнить

Практически вся базовая работа с JSON-массивами сводится к нескольким методам:

```text
find()       → найти один объект
filter()     → получить несколько объектов
map()        → преобразовать объекты
sort()       → отсортировать
findIndex()  → найти индекс
some()       → есть ли хотя бы один
every()      → подходят ли все
reduce()     → получить одно итоговое значение
push()       → добавить элемент
```

А во Vue чаще всего схема выглядит так:

```text
JSON / API
    ↓
массив объектов
    ↓
ref()
    ↓
computed()
    ↓
filter / map / sort / find
    ↓
шаблон Vue
```

Самая важная идея: **JSON — это просто данные.** После того как они попали в приложение, мы работаем с ними обычными возможностями JavaScript/TypeScript, а Vue отвечает в основном за реактивность и отображение этих данных.
