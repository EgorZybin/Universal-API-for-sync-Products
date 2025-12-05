# Product Sync API Documentation

Универсальный REST API для синхронизации товаров из внешних складских систем.

## Оглавление

- [Базовый URL](#базовый-url)
- [Аутентификация](#аутентификация)
- [Endpoint](#endpoint)
- [Структура запроса](#структура-запроса)
- [Описание полей](#описание-полей)
- [Примеры запросов](#примеры-запросов)
- [Ответы API](#ответы-api)
- [Обработка ошибок](#обработка-ошибок)
- [Логика работы](#логика-работы)

---

## Базовый URL

```
https://your-domain.com/webapp/api/sync/product
```

## Аутентификация

API использует простую аутентификацию через API ключ. Ключ должен быть указан в переменных окружения сервера как `PRODUCT_SYNC_API_KEY`.

### Способы передачи ключа:

**Вариант 1: Заголовок X-API-Key**
```
X-API-Key: your_secret_key_here
```

**Вариант 2: Bearer токен в Authorization**
```
Authorization: Bearer your_secret_key_here
```

### Получение API ключа

Обратитесь к администратору системы для получения API ключа.

---

## Endpoint

### Синхронизация товара

**POST** `/webapp/api/sync/product`

Создает новый товар или обновляет существующий (если передан `external_id`).

---

## Структура запроса

### Минимальный запрос

```json
{
  "name": "Название товара"
}
```

### Полный запрос

```json
{
  "name": "Название товара",
  "external_id": "SKU-12345",
  "on_main": false,
  "images": [...],
  "categories": [...],
  "parameters": [...],
  "colors": [...],
  "specifications": [...],
  "extra": {...},
  "tags": [...],
  "quantity": [...],
  "extra_json": {...}
}
```

---

## Описание полей

### Основные поля

| Поле | Тип | Обязательное | Описание |
|------|-----|---------------|----------|
| `name` | `string` | ✅ Да | Название товара (1-255 символов) |
| `external_id` | `string` | ❌ Нет | Уникальный идентификатор товара во внешней системе. Используется для обновления существующих товаров |
| `on_main` | `boolean` | ❌ Нет | Показывать товар на главной странице (по умолчанию `false`) |
| `extra_json` | `object` | ❌ Нет | Дополнительные данные в формате JSON |

### Изображения (`images`)

Массив объектов изображений товара. Поддерживается два способа передачи изображений:

1. **URL изображения** - для использования внешних ссылок
2. **Base64 данные** - для загрузки изображений напрямую в S3 хранилище

**Важно:** В каждом объекте изображения должно быть указано либо `url`, либо `base64_data` (хотя бы одно из полей).

#### Способ 1: URL изображения

```json
{
  "images": [
    {
      "url": "https://example.com/image1.jpg",
      "main": true,
      "title": "Главное изображение",
      "position": "front",
      "sort_order": 0
    }
  ]
}
```

#### Способ 2: Base64 данные (загрузка в S3)

```json
{
  "images": [
    {
      "base64_data": "data:image/jpeg;base64,/9j/4AAQSkZJRgABAQEAYABgAAD...",
      "main": true,
      "title": "Главное изображение",
      "sort_order": 0
    }
  ]
}
```

**Форматы base64:**
- С префиксом data URI: `"data:image/jpeg;base64,/9j/4AAQ..."`
- Чистый base64: `"/9j/4AAQ..."` (автоматически определится как JPEG)

**Что происходит при использовании base64:**
1. Изображение декодируется из base64
2. Автоматически загружается в S3 хранилище
3. Сохраняется по пути: `product_sync_images/{external_id}/{uuid}.{ext}`
4. Изображение автоматически сжимается (создаются оптимизированные версии)
5. В базу данных сохраняется URL загруженного изображения

| Поле | Тип | Обязательное | Описание |
|------|-----|---------------|----------|
| `url` | `string` | ❌ Нет* | URL изображения (внешняя ссылка) |
| `base64_data` | `string` | ❌ Нет* | Base64 строка изображения (для загрузки в S3) |
| `main` | `boolean` | ❌ Нет | Главное изображение (по умолчанию `false`) |
| `title` | `string` | ❌ Нет | Заголовок изображения |
| `position` | `string` | ❌ Нет | Позиция изображения |
| `sort_order` | `integer` | ❌ Нет | Порядок сортировки (по умолчанию `0`) |

\* **Обязательно указать хотя бы одно из полей:** `url` или `base64_data`

**Рекомендации:**
- Используйте `base64_data` для интеграций из 1C и других систем, которые не могут предоставить публичные URL
- Используйте `url` для внешних изображений, которые уже доступны по ссылке
- Можно комбинировать оба способа в одном массиве изображений
- Для больших изображений (>5MB) рекомендуется предварительное сжатие на стороне клиента

### Категории (`categories`)

Массив категорий. Поддерживается три формата:

1. **ID категории (integer)** - для использования существующих категорий
2. **Название категории (string)** - категория создастся автоматически, если не существует
3. **Объект категории (object)** - создание категории с полной информацией (родитель, изображение и т.д.)

```json
{
  "categories": [
    1,
    "Простая категория",
    {
      "name": "Категория с полной информацией",
      "parent_id": 1,
      "image": "https://example.com/category.jpg",
      "sort_order": 10
    }
  ]
}
```

#### Формат 1: ID категории (integer)

```json
{
  "categories": [1, 2, 3]
}
```

- Используется для привязки к существующим категориям
- Если категория с указанным ID не найдена, она будет пропущена
- **Рекомендуется:** Получайте актуальные ID через `GET /webapp/api/sync/categories`

#### Формат 2: Название категории (string)

```json
{
  "categories": ["Одежда", "Джинсы"]
}
```

- Категория ищется по названию
- Если категория не найдена, она создается автоматически с минимальными данными
- Подходит для простых случаев

#### Формат 3: Объект категории (object)

```json
{
  "categories": [
    {
      "name": "Новая категория",
      "parent_id": 1,
      "parent_name": "Одежда",
      "image": "https://example.com/category.jpg",
      "sort_order": 5
    }
  ]
}
```

| Поле | Тип | Обязательное | Описание |
|------|-----|---------------|----------|
| `name` | `string` | ✅ Да | Название категории |
| `parent_id` | `integer` | ❌ Нет | ID родительской категории |
| `parent_name` | `string` | ❌ Нет | Название родительской категории (будет создана, если не существует) |
| `image` | `string` | ❌ Нет | URL изображения категории |
| `sort_order` | `integer` | ❌ Нет | Порядок сортировки (по умолчанию `0`) |

**Логика работы:**
- Категория ищется по названию (`name`)
- Если категория не найдена, она создается с указанными параметрами
- Если категория найдена, она обновляется (родитель, изображение, порядок сортировки)
- Родительская категория определяется по `parent_id` или `parent_name`
- Если указано `parent_name` и категория не найдена, родительская категория создается автоматически

**Важно:** 
- Можно комбинировать все три формата в одном массиве
- При использовании объектов категорий можно создавать иерархию категорий "на лету"
- Для получения актуального списка категорий используйте `GET /webapp/api/sync/categories`

### Параметры (`parameters`)

Массив параметров товара (варианты, размеры, модификации). Если параметры не указаны, будет создан один параметр по умолчанию.

```json
{
  "parameters": [
    {
      "name": "Варианты",
      "parameter_string": "Размер M",
      "price": 1000,
      "old_price": 1200,
      "disabled": false,
      "chosen": true,
      "extra_field_color": "#FF0000",
      "extra_field_image": "https://example.com/variant.jpg",
      "sort_order": 0,
      "parameter_json": {
        "custom_field": "value"
      }
    }
  ]
}
```

| Поле | Тип | Обязательное | Описание |
|------|-----|---------------|----------|
| `name` | `string` | ❌ Нет | Название параметра (по умолчанию "Варианты") |
| `parameter_string` | `string` | ❌ Нет | Строковое значение параметра (по умолчанию название товара) |
| `price` | `integer` | ✅ Да | Цена в копейках |
| `old_price` | `integer` | ❌ Нет | Старая цена в копейках |
| `disabled` | `boolean` | ❌ Нет | Отключен ли параметр (по умолчанию `false`) |
| `chosen` | `boolean` | ❌ Нет | Выбран по умолчанию (по умолчанию `true`) |
| `extra_field_color` | `string` | ❌ Нет | Дополнительное поле - цвет |
| `extra_field_image` | `string` | ❌ Нет | Дополнительное поле - изображение |
| `sort_order` | `integer` | ❌ Нет | Порядок сортировки (по умолчанию `0`) |
| `parameter_json` | `object` | ❌ Нет | Дополнительные данные в формате JSON |

### Цвета (`colors`)

Массив цветов товара.

```json
{
  "colors": [
    {
      "name": "Красный",
      "code": "#FF0000",
      "image": "https://example.com/color.jpg",
      "discount": 10,
      "sort_order": 0,
      "json_data": {
        "custom": "data"
      }
    }
  ]
}
```

| Поле | Тип | Обязательное | Описание |
|------|-----|---------------|----------|
| `name` | `string` | ✅ Да | Название цвета |
| `code` | `string` | ✅ Да | HEX код цвета |
| `image` | `string` | ❌ Нет | URL изображения цвета |
| `discount` | `integer` | ❌ Нет | Скидка в процентах |
| `sort_order` | `integer` | ❌ Нет | Порядок сортировки |
| `json_data` | `object` | ❌ Нет | Дополнительные данные в формате JSON |

### Характеристики (`specifications`)

Массив характеристик товара.

```json
{
  "specifications": [
    {
      "name": "Материал",
      "type": "TEXT",
      "text_value": "Хлопок 100%",
      "is_for_filter": true,
      "is_visible": true
    },
    {
      "name": "Вес",
      "type": "NUMERIC",
      "numeric_value": 500,
      "unit_measurement_id": 1,
      "is_for_filter": true,
      "is_visible": true
    }
  ]
}
```

| Поле | Тип | Обязательное | Описание |
|------|-----|---------------|----------|
| `name` | `string` | ✅ Да | Название характеристики |
| `type` | `string` | ❌ Нет | Тип: `"TEXT"` или `"NUMERIC"` (по умолчанию `"TEXT"`) |
| `text_value` | `string` | ❌ Нет | Текстовое значение (для типа TEXT) |
| `numeric_value` | `integer` | ❌ Нет | Числовое значение (для типа NUMERIC) |
| `unit_measurement_id` | `integer` | ❌ Нет | ID единицы измерения |
| `is_for_filter` | `boolean` | ❌ Нет | Использовать в фильтрах (по умолчанию `true`) |
| `is_visible` | `boolean` | ❌ Нет | Видимость (по умолчанию `true`) |

### Дополнительная информация (`extra`)

Дополнительная информация о товаре.

```json
{
  "extra": {
    "characteristics": "Описание характеристик",
    "kit": "Комплектация",
    "offer": "Предложение",
    "delivery": "Информация о доставке",
    "ai_description": "AI описание"
  }
}
```

| Поле | Тип | Обязательное | Описание |
|------|-----|---------------|----------|
| `characteristics` | `string` | ❌ Нет | Характеристики товара |
| `kit` | `string` | ❌ Нет | Комплектация |
| `offer` | `string` | ❌ Нет | Предложение |
| `delivery` | `string` | ❌ Нет | Информация о доставке |
| `ai_description` | `string` | ❌ Нет | AI описание |

### Теги (`tags`)

Массив тегов товара.

```json
{
  "tags": ["новинка", "хит продаж", "скидка"]
}
```

| Тип | Описание |
|-----|----------|
| `array<string>` | Массив строк-тегов |

### Количество (`quantity`)

Массив записей о количестве товара по параметрам и цветам.

```json
{
  "quantity": [
    {
      "parameter_index": 0,
      "color_index": 0,
      "stock": 10,
      "reserve": 2,
      "in_transit": 5,
      "sold": 0,
      "is_infinite": false,
      "additional_info": {
        "warehouse": "Moscow"
      }
    }
  ]
}
```

| Поле | Тип | Обязательное | Описание |
|------|-----|---------------|----------|
| `parameter_index` | `integer` | ❌ Нет | Индекс параметра из массива `parameters` (по умолчанию `0`) |
| `color_index` | `integer` | ❌ Нет | Индекс цвета из массива `colors` (по умолчанию `null`) |
| `stock` | `integer` | ✅ Да | Физический остаток на складах |
| `reserve` | `integer` | ❌ Нет | В резерве (по умолчанию `0`) |
| `in_transit` | `integer` | ❌ Нет | Ожидание поступления (по умолчанию `0`) |
| `sold` | `integer` | ❌ Нет | Продано (по умолчанию `0`) |
| `is_infinite` | `boolean` | ❌ Нет | Неограниченное количество (по умолчанию `false`) |
| `additional_info` | `object` | ❌ Нет | Дополнительная информация в формате JSON |

**Важно:** `parameter_index` и `color_index` указывают на позицию элемента в массивах `parameters` и `colors` соответственно (начинается с 0).

---

## Примеры запросов

### Пример 0: Создание товара с новой категорией

**Самый простой способ - создание товара с новой категорией по названию:**

```bash
curl -X POST https://your-domain.com/webapp/api/sync/product \
  -H "X-API-Key: your_secret_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Футболка премиум",
    "external_id": "T-SHIRT-PREMIUM-001",
    "images": [
      {
        "url": "https://example.com/tshirt.jpg",
        "main": true
      }
    ],
    "categories": ["Премиум одежда"],
    "parameters": [
      {
        "name": "Размеры",
        "parameter_string": "M",
        "price": 500000
      }
    ],
    "quantity": [
      {
        "parameter_index": 0,
        "stock": 10,
        "is_infinite": false
      }
    ]
  }'
```

**Создание товара с новой категорией и полной информацией (изображение, порядок сортировки):**

```bash
curl -X POST https://your-domain.com/webapp/api/sync/product \
  -H "X-API-Key: your_secret_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Джинсы классические",
    "external_id": "JEANS-CLASSIC-001",
    "images": [
      {
        "url": "https://example.com/jeans.jpg",
        "main": true
      }
    ],
    "categories": [
      {
        "name": "Одежда",
        "image": "https://example.com/clothing.jpg",
        "sort_order": 1
      },
      {
        "name": "Джинсы",
        "parent_name": "Одежда",
        "image": "https://example.com/jeans-category.jpg",
        "sort_order": 2
      }
    ],
    "parameters": [
      {
        "name": "Размеры",
        "parameter_string": "32",
        "price": 300000
      },
      {
        "name": "Размеры",
        "parameter_string": "34",
        "price": 300000
      }
    ],
    "quantity": [
      {
        "parameter_index": 0,
        "stock": 10,
        "is_infinite": false
      },
      {
        "parameter_index": 1,
        "stock": 15,
        "is_infinite": false
      }
    ]
  }'
```

**Создание товара с иерархией категорий (родитель → дочерняя → подкатегория):**

```bash
curl -X POST https://your-domain.com/webapp/api/sync/product \
  -H "X-API-Key: your_secret_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Кроссовки беговые",
    "external_id": "SNEAKERS-RUNNING-001",
    "images": [
      {
        "url": "https://example.com/sneakers.jpg",
        "main": true
      }
    ],
    "categories": [
      {
        "name": "Обувь",
        "sort_order": 1
      },
      {
        "name": "Спортивная обувь",
        "parent_name": "Обувь",
        "sort_order": 2
      },
      {
        "name": "Беговая обувь",
        "parent_name": "Спортивная обувь",
        "image": "https://example.com/running-shoes.jpg",
        "sort_order": 3
      }
    ],
    "parameters": [
      {
        "name": "Размеры",
        "parameter_string": "42",
        "price": 800000,
        "old_price": 1000000
      }
    ],
    "quantity": [
      {
        "parameter_index": 0,
        "stock": 20,
        "is_infinite": false
      }
    ],
    "specifications": [
      {
        "name": "Материал",
        "type": "TEXT",
        "text_value": "Синтетика, сетка"
      },
      {
        "name": "Вес",
        "type": "NUMERIC",
        "numeric_value": 250
      }
    ],
    "extra": {
      "characteristics": "Легкие беговые кроссовки для тренировок",
      "kit": "Кроссовки, коробка"
    }
  }'
```

**Что происходит при выполнении запроса:**
1. Создается товар "Кроссовки беговые"
2. Создается категория "Обувь" (если не существует)
3. Создается категория "Спортивная обувь" с родителем "Обувь" (если не существует)
4. Создается категория "Беговая обувь" с родителем "Спортивная обувь" (если не существует)
5. Товар привязывается ко всем трем категориям
6. Создаются параметры, количество, характеристики и дополнительная информация

### Пример 1: Простой товар

```bash
curl -X POST https://your-domain.com/webapp/api/sync/product \
  -H "X-API-Key: your_secret_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Футболка",
    "external_id": "T-SHIRT-001"
  }'
```

### Пример 1.1: Создание товара с новой категорией

**Создание товара и новой категории одновременно:**

```bash
curl -X POST https://your-domain.com/webapp/api/sync/product \
  -H "X-API-Key: your_secret_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Футболка премиум",
    "external_id": "T-SHIRT-PREMIUM-001",
    "images": [
      {
        "url": "https://example.com/tshirt.jpg",
        "main": true
      }
    ],
    "categories": [
      {
        "name": "Премиум одежда",
        "image": "https://example.com/premium-category.jpg",
        "sort_order": 1
      }
    ],
    "parameters": [
      {
        "name": "Размеры",
        "parameter_string": "M",
        "price": 500000,
        "old_price": 600000
      }
    ]
  }'
```

**Создание товара с иерархией категорий:**

```bash
curl -X POST https://your-domain.com/webapp/api/sync/product \
  -H "X-API-Key: your_secret_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Джинсы классические",
    "external_id": "JEANS-CLASSIC-001",
    "images": [
      {
        "url": "https://example.com/jeans.jpg",
        "main": true
      }
    ],
    "categories": [
      {
        "name": "Одежда",
        "sort_order": 1
      },
      {
        "name": "Джинсы",
        "parent_name": "Одежда",
        "image": "https://example.com/jeans-category.jpg",
        "sort_order": 2
      },
      {
        "name": "Классические",
        "parent_name": "Джинсы",
        "sort_order": 3
      }
    ],
    "parameters": [
      {
        "name": "Размеры",
        "parameter_string": "32",
        "price": 300000
      },
      {
        "name": "Размеры",
        "parameter_string": "34",
        "price": 300000
      }
    ],
    "quantity": [
      {
        "parameter_index": 0,
        "stock": 10,
        "is_infinite": false
      },
      {
        "parameter_index": 1,
        "stock": 15,
        "is_infinite": false
      }
    ]
  }'
```

**Создание товара с категорией, у которой есть родитель по ID:**

```bash
curl -X POST https://your-domain.com/webapp/api/sync/product \
  -H "X-API-Key: your_secret_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Кроссовки спортивные",
    "external_id": "SNEAKERS-SPORT-001",
    "images": [
      {
        "url": "https://example.com/sneakers.jpg",
        "main": true
      }
    ],
    "categories": [
      {
        "name": "Спортивная обувь",
        "parent_id": 5,
        "image": "https://example.com/sport-shoes.jpg",
        "sort_order": 10
      }
    ],
    "parameters": [
      {
        "name": "Размеры",
        "parameter_string": "42",
        "price": 800000
      }
    ],
    "quantity": [
      {
        "parameter_index": 0,
        "stock": 20,
        "is_infinite": false
      }
    ]
  }'
```

### Пример 2: Товар с изображениями и категориями

**С использованием ID категорий (рекомендуется):**
```bash
curl -X POST https://your-domain.com/webapp/api/sync/product \
  -H "X-API-Key: your_secret_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Джинсы",
    "external_id": "JEANS-001",
    "on_main": true,
    "images": [
      {
        "url": "https://example.com/jeans1.jpg",
        "main": true,
        "sort_order": 0
      },
      {
        "url": "https://example.com/jeans2.jpg",
        "main": false,
        "sort_order": 1
      }
    ],
    "categories": [1, 2]
  }'
```

**С использованием base64 изображений (для интеграций из 1C):**
```bash
curl -X POST https://your-domain.com/webapp/api/sync/product \
  -H "X-API-Key: your_secret_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Джинсы",
    "external_id": "JEANS-001",
    "on_main": true,
    "images": [
      {
        "base64_data": "data:image/jpeg;base64,/9j/4AAQSkZJRgABAQEAYABgAAD...",
        "main": true,
        "title": "Главное изображение",
        "sort_order": 0
      },
      {
        "base64_data": "data:image/jpeg;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAAB...",
        "main": false,
        "title": "Дополнительное изображение",
        "sort_order": 1
      }
    ],
    "categories": [1, 2]
  }'
```

**Комбинированный подход (URL + base64):**
```bash
curl -X POST https://your-domain.com/webapp/api/sync/product \
  -H "X-API-Key: your_secret_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Джинсы",
    "external_id": "JEANS-001",
    "on_main": true,
    "images": [
      {
        "base64_data": "data:image/jpeg;base64,/9j/4AAQSkZJRgABAQEAYABgAAD...",
        "main": true,
        "sort_order": 0
      },
      {
        "url": "https://example.com/jeans2.jpg",
        "main": false,
        "sort_order": 1
      }
    ],
    "categories": [1, 2]
  }'
```

**С использованием названий категорий (категории создадутся автоматически, если не существуют):**
```bash
curl -X POST https://your-domain.com/webapp/api/sync/product \
  -H "X-API-Key: your_secret_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Джинсы",
    "external_id": "JEANS-001",
    "on_main": true,
    "images": [
      {
        "url": "https://example.com/jeans1.jpg",
        "main": true,
        "sort_order": 0
      }
    ],
    "categories": ["Одежда", "Джинсы"]
  }'
```

**С созданием категорий через объекты (с полной информацией):**
```bash
curl -X POST https://your-domain.com/webapp/api/sync/product \
  -H "X-API-Key: your_secret_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Джинсы",
    "external_id": "JEANS-001",
    "on_main": true,
    "images": [
      {
        "url": "https://example.com/jeans1.jpg",
        "main": true
      }
    ],
    "categories": [
      {
        "name": "Одежда",
        "image": "https://example.com/clothing.jpg",
        "sort_order": 1
      },
      {
        "name": "Джинсы",
        "parent_name": "Одежда",
        "image": "https://example.com/jeans-category.jpg",
        "sort_order": 2
      }
    ]
  }'
```

**Комбинированный подход (ID + объекты):**
```bash
curl -X POST https://your-domain.com/webapp/api/sync/product \
  -H "X-API-Key: your_secret_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Джинсы",
    "external_id": "JEANS-001",
    "categories": [
      1,
      {
        "name": "Новая подкатегория",
        "parent_id": 1,
        "sort_order": 10
      }
    ]
  }'
```

### Пример 3: Товар с параметрами и количеством

```bash
curl -X POST https://your-domain.com/webapp/api/sync/product \
  -H "X-API-Key: your_secret_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Кроссовки",
    "external_id": "SNEAKERS-001",
    "parameters": [
      {
        "name": "Размеры",
        "parameter_string": "42",
        "price": 500000,
        "old_price": 600000
      },
      {
        "name": "Размеры",
        "parameter_string": "43",
        "price": 500000,
        "old_price": 600000
      }
    ],
    "quantity": [
      {
        "parameter_index": 0,
        "stock": 5,
        "is_infinite": false
      },
      {
        "parameter_index": 1,
        "stock": 10,
        "is_infinite": false
      }
    ]
  }'
```

### Пример 4: Полный товар со всеми полями

```bash
curl -X POST https://your-domain.com/webapp/api/sync/product \
  -H "X-API-Key: your_secret_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Платье летнее",
    "external_id": "DRESS-001",
    "on_main": true,
    "images": [
      {
        "url": "https://example.com/dress1.jpg",
        "main": true
      }
    ],
    "categories": [1, "Летняя коллекция"],
    "parameters": [
      {
        "name": "Размеры",
        "parameter_string": "S",
        "price": 300000,
        "old_price": 350000
      },
      {
        "name": "Размеры",
        "parameter_string": "M",
        "price": 300000,
        "old_price": 350000
      }
    ],
    "colors": [
      {
        "name": "Белый",
        "code": "#FFFFFF"
      },
      {
        "name": "Черный",
        "code": "#000000"
      }
    ],
    "specifications": [
      {
        "name": "Материал",
        "type": "TEXT",
        "text_value": "Хлопок 100%"
      },
      {
        "name": "Длина",
        "type": "NUMERIC",
        "numeric_value": 95,
        "unit_measurement_id": 1
      }
    ],
    "extra": {
      "characteristics": "Легкое летнее платье из натурального хлопка",
      "kit": "Платье",
      "delivery": "Доставка 1-3 дня"
    },
    "tags": ["новинка", "лето"],
    "quantity": [
      {
        "parameter_index": 0,
        "color_index": 0,
        "stock": 10,
        "is_infinite": false
      },
      {
        "parameter_index": 0,
        "color_index": 1,
        "stock": 5,
        "is_infinite": false
      },
      {
        "parameter_index": 1,
        "color_index": 0,
        "stock": 8,
        "is_infinite": false
      },
      {
        "parameter_index": 1,
        "color_index": 1,
        "stock": 3,
        "is_infinite": false
      }
    ],
    "extra_json": {
      "source": "warehouse_system",
      "barcode": "1234567890123"
    }
  }'
```

### Пример 5: Обновление существующего товара

```bash
curl -X POST https://your-domain.com/webapp/api/sync/product \
  -H "X-API-Key: your_secret_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Обновленное название",
    "external_id": "T-SHIRT-001",
    "parameters": [
      {
        "name": "Варианты",
        "parameter_string": "Обновленный вариант",
        "price": 150000
      }
    ],
    "quantity": [
      {
        "parameter_index": 0,
        "stock": 20
      }
    ]
  }'
```

---

## Ответы API

### Успешный ответ

**HTTP Status:** `200 OK`

```json
{
  "status": "ok",
  "message": "Product synced successfully",
  "product_id": 123,
  "external_id": "SKU-12345"
}
```

| Поле | Тип | Описание |
|------|-----|----------|
| `status` | `string` | Статус ответа (`"ok"`) |
| `message` | `string` | Сообщение |
| `product_id` | `integer` | ID созданного/обновленного товара |
| `external_id` | `string` | Внешний ID товара |

### Ошибка валидации

**HTTP Status:** `400 Bad Request`

```json
{
  "status": "error",
  "message": "Product name must be between 1 and 255 characters"
}
```

### Ошибка аутентификации

**HTTP Status:** `401 Unauthorized`

```json
{
  "status": "error",
  "message": "Invalid API key"
}
```

### Ошибка сервера

**HTTP Status:** `500 Internal Server Error`

```json
{
  "status": "error",
  "message": "Database error: ..."
}
```

---

## Обработка ошибок

### Коды ошибок

| HTTP Status | Описание |
|-------------|----------|
| `400` | Ошибка валидации данных |
| `401` | Неверный API ключ |
| `500` | Внутренняя ошибка сервера |

### Типичные ошибки

**Ошибка: "Invalid API key"**
- Проверьте правильность API ключа
- Убедитесь, что ключ передается в заголовке `X-API-Key` или `Authorization`

**Ошибка: "Request body is required"**
- Убедитесь, что тело запроса содержит JSON
- Проверьте заголовок `Content-Type: application/json`

**Ошибка: "Product name must be between 1 and 255 characters"**
- Название товара должно быть от 1 до 255 символов
- Название не может быть пустым

**Ошибка: "Database error: ..."**
- Внутренняя ошибка базы данных
- Обратитесь к администратору системы

---

## Логика работы

### Создание нового товара

1. Если `external_id` не указан или товар с таким `external_id` не найден, создается новый товар
2. Все переданные данные сохраняются
3. Если параметры не указаны, создается один параметр по умолчанию
4. Возвращается `product_id` созданного товара

### Обновление существующего товара

1. Если указан `external_id` и товар с таким ID существует, товар обновляется
2. Обновляются только переданные поля:
   - Если переданы `images`, старые изображения удаляются и создаются новые
   - Если переданы `categories`, старые категории удаляются и добавляются новые
   - Если переданы `parameters`, старые параметры удаляются и создаются новые
   - Если переданы `colors`, старые цвета удаляются и создаются новые
   - Если переданы `specifications`, старые характеристики удаляются и создаются новые
   - Если переданы `tags`, старые теги удаляются и создаются новые
   - Если переданы `quantity`, старые записи количества удаляются и создаются новые
3. Поля, которые не переданы, остаются без изменений
4. Возвращается `product_id` обновленного товара

### Работа с категориями

**Получение списка категорий:**
- Используйте endpoint `GET /webapp/api/sync/categories` для получения всех категорий с их ID
- Это рекомендуется делать перед созданием товаров для использования актуальных ID

**Три способа работы с категориями:**

1. **Использование ID категорий (рекомендуется для существующих):**
   - Передавайте массив чисел: `"categories": [1, 2, 3]`
   - Если категория с указанным ID не найдена, она будет пропущена
   - Самый надежный способ для существующих категорий

2. **Использование названий категорий (простое создание):**
   - Передавайте массив строк: `"categories": ["Одежда", "Джинсы"]`
   - Категория ищется по названию
   - Если не найдена, создается автоматически с минимальными данными

3. **Использование объектов категорий (создание с полной информацией):**
   - Передавайте массив объектов: `"categories": [{"name": "Категория", "parent_id": 1, "image": "url"}]`
   - Позволяет создавать категории с родителями, изображениями, порядком сортировки
   - Если категория существует, она обновляется
   - Можно создавать иерархию категорий "на лету"

**Смешанное использование:**
- Можно комбинировать все три формата: `"categories": [1, "Простая категория", {"name": "Сложная категория", "parent_id": 1}]`
- Это позволяет использовать существующие категории по ID и создавать новые с нужными параметрами

### Обработка параметров

- Если параметры не указаны, создается один параметр по умолчанию:
  - `name`: "Варианты"
  - `parameter_string`: название товара
  - `price`: 0
  - `chosen`: true

### Обработка количества

- `parameter_index` указывает на позицию параметра в массиве `parameters` (начинается с 0)
- `color_index` указывает на позицию цвета в массиве `colors` (начинается с 0)
- Если `parameter_index` не указан или неверный, используется первый параметр (индекс 0)
- Если `color_index` не указан или неверный, количество привязывается без цвета

### Кэширование

После успешной синхронизации товара весь кэш системы автоматически очищается для обеспечения актуальности данных.

---

## Рекомендации

1. **Всегда используйте `external_id`** для идентификации товаров во внешней системе
2. **Передавайте полные данные** при первом создании товара
3. **Используйте индексы правильно** в массиве `quantity` - они должны соответствовать позициям в `parameters` и `colors`
4. **Обрабатывайте ошибки** и логируйте их для отладки
5. **Используйте idempotency** - повторный запрос с теми же данными не должен создавать дубликаты
6. **Для изображений:**
   - Используйте `base64_data` для интеграций из 1C и систем, которые не могут предоставить публичные URL
   - Используйте `url` для внешних изображений, которые уже доступны по ссылке
   - Можно комбинировать оба способа в одном массиве изображений
   - Для больших изображений (>5MB) рекомендуется предварительное сжатие на стороне клиента
   - Base64 увеличивает размер данных на ~33%, учитывайте лимиты размера HTTP запросов

---

## Поддержка

При возникновении проблем обращайтесь к администратору системы или в техническую поддержку.

## Дополнительные материалы

### Работа с base64 изображениями

**Для разработчиков 1C:**
- См. файл `1C_BASE64_EXAMPLES.md` с примерами кода для конвертации изображений в base64 в 1С

**Для тестирования в Postman:**
- См. файл `POSTMAN_TEST_EXAMPLES.md` с примерами запросов и способами получения base64 из изображений

**Форматы base64:**
- С префиксом data URI: `"data:image/jpeg;base64,/9j/4AAQ..."`
- Чистый base64: `"/9j/4AAQ..."` (автоматически определится как JPEG)

**Что происходит при загрузке:**
1. Изображение декодируется из base64
2. Автоматически загружается в S3 хранилище
3. Сохраняется по пути: `product_sync_images/{external_id}/{uuid}.{ext}`
4. Изображение автоматически сжимается (создаются оптимизированные версии)
5. В базу данных сохраняется URL загруженного изображения

**Версия API:** 1.1  
**Последнее обновление:** 2024

