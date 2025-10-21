# Search Filter - Документация полнотекстового поиска

## Описание

`SearchFilter` - это специализированный фильтр для полнотекстового поиска по нескольким полям одновременно. Поддерживает
поиск как по простым полям модели, так и по полям связанных моделей через точечную нотацию.

---

## Архитектура

### Путь обработки

```
Request → CustomQueryBuilder::applySearch() 
    → SearchFilter::apply()
        → RelationFieldParser::parse() (для каждого поля)
            → applySearchDirect() или applySearchWithRelation()
```

### Основные компоненты

1. **SearchFilter** (`app/Support/QueryBuilders/Filters/SearchFilter.php`)
    - Основной класс обработки поиска
    - Обрабатывает массив полей для поиска
    - Автоматически определяет наличие relations

2. **RelationFieldParser** (`app/Support/QueryBuilders/Filters/Support/RelationFieldParser.php`)
    - Парсит поля с точечной нотацией
    - Разделяет relation path и конечное поле
    - Пример: `user.profile.city` → relations: `['user', 'profile']`, field: `'city'`

---

## Формат запроса

### Базовая структура

```json
{
    "search": {
        "query": "значение для поиска",
        "fields": [
            "поле1",
            "поле2",
            "relation.поле3"
        ]
    }
}
```

### Параметры

| Параметр | Тип              | Обязательный    | Описание                |
|----------|------------------|-----------------|-------------------------|
| `query`  | `string\|number` | ✅ Да            | Поисковый запрос        |
| `value`  | `string\|number` | ⚠️ Альтернатива | Алиас для `query`       |
| `fields` | `array`          | ✅ Да            | Массив полей для поиска |

---

## Логика поиска

### Для строковых значений

```sql
WHERE (field ILIKE '%query%')
```

- Использует **ILIKE** (регистронезависимый поиск в PostgreSQL)
- Автоматически добавляет wildcards `%` с обеих сторон
- Поиск вхождения подстроки в любом месте поля

### Для числовых значений

```sql
WHERE (field ILIKE '%query%' OR field = query)
```

- Выполняет **два условия**:
    1. `ILIKE` поиск (для частичного совпадения, например: "123" найдет "12345")
    2. Точное совпадение `=` (для полного совпадения числа)

### Комбинация полей

Все поля объединяются через **OR**:

```sql
WHERE (
  field1 ILIKE '%query%' 
  OR field2 ILIKE '%query%' 
  OR field3 ILIKE '%query%'
)
```

---

## Работа с Relations

### Автоматическое определение

Система автоматически определяет relations по наличию точки в имени поля:

```
"user.email" → relation detected
"email"      → direct field
```

### Обработка через whereHas

Для полей с relations используется **Laravel orWhereHas**:

```php
$builder->orWhereHas('relationName', function($query) use ($field, $searchQuery) {
    $query->where($field, 'ILIKE', "%{$searchQuery}%");
    if (is_numeric($searchQuery)) {
        $query->orWhere($field, '=', $searchQuery);
    }
});
```

### Вложенные Relations

Поддерживается неограниченная вложенность:

```json
{
    "search": {
        "query": "Moscow",
        "fields": [
            "user.profile.city.name"
        ]
    }
}
```

Преобразуется в:

```sql
OR EXISTS (
  SELECT * FROM users
  WHERE table.user_id = users.id
  AND EXISTS (
    SELECT * FROM profiles
    WHERE users.profile_id = profiles.id
    AND EXISTS (
      SELECT * FROM cities
      WHERE profiles.city_id = cities.id
      AND cities.name ILIKE '%Moscow%'
    )
  )
)
```

---

## Примеры использования

### 1. Простой поиск по одному полю

**Запрос:**

```json
{
    "search": {
        "query": "laptop",
        "fields": [
            "name"
        ]
    }
}
```

**SQL:**

```sql
WHERE (name ILIKE '%laptop%')
```

---

### 2. Поиск по нескольким полям

**Запрос:**

```json
{
    "search": {
        "query": "apple",
        "fields": [
            "name",
            "description",
            "brand"
        ]
    }
}
```

**SQL:**

```sql
WHERE (
  name ILIKE '%apple%'
  OR description ILIKE '%apple%'
  OR brand ILIKE '%apple%'
)
```

---

### 3. Числовой поиск

**Запрос:**

```json
{
    "search": {
        "query": "1000",
        "fields": [
            "price",
            "sku"
        ]
    }
}
```

**SQL:**

```sql
WHERE (
  (price ILIKE '%1000%' OR price = 1000)
  OR (sku ILIKE '%1000%' OR sku = 1000)
)
```

**Результаты:**

- Товары с ценой ровно 1000
- Товары с ценой 10000, 21000 (содержат "1000")
- SKU вида "ABC-1000-XYZ" или "1000"

---

### 4. Поиск по relation (одна вложенность)

**Запрос:**

```json
{
    "search": {
        "query": "electronics",
        "fields": [
            "name",
            "category.name"
        ]
    }
}
```

**SQL:**

```sql
WHERE (
  name ILIKE '%electronics%'
  OR EXISTS (
    SELECT * FROM categories
    WHERE products.category_id = categories.id
    AND categories.name ILIKE '%electronics%'
  )
)
```

---

### 5. Поиск по глубоким relations

**Запрос:**

```json
{
    "search": {
        "query": "john",
        "fields": [
            "title",
            "author.name",
            "author.profile.bio",
            "publisher.city.name"
        ]
    }
}
```

**SQL:**

```sql
WHERE (
  title ILIKE '%john%'
  OR EXISTS (
    SELECT * FROM authors
    WHERE books.author_id = authors.id
    AND authors.name ILIKE '%john%'
  )
  OR EXISTS (
    SELECT * FROM authors
    WHERE books.author_id = authors.id
    AND EXISTS (
      SELECT * FROM profiles
      WHERE authors.profile_id = profiles.id
      AND profiles.bio ILIKE '%john%'
    )
  )
  OR EXISTS (
    SELECT * FROM publishers
    WHERE books.publisher_id = publishers.id
    AND EXISTS (
      SELECT * FROM cities
      WHERE publishers.city_id = cities.id
      AND cities.name ILIKE '%john%'
    )
  )
)
```

---

### 6. Поиск email адресов

**Запрос:**

```json
{
    "search": {
        "query": "@gmail.com",
        "fields": [
            "email",
            "user.email",
            "contact.email"
        ]
    }
}
```

**SQL:**

```sql
WHERE (
  email ILIKE '%@gmail.com%'
  OR EXISTS (
    SELECT * FROM users
    WHERE records.user_id = users.id
    AND users.email ILIKE '%@gmail.com%'
  )
  OR EXISTS (
    SELECT * FROM contacts
    WHERE records.contact_id = contacts.id
    AND contacts.email ILIKE '%@gmail.com%'
  )
)
```

---

### 7. Поиск по телефону

**Запрос:**

```json
{
    "search": {
        "query": "+7",
        "fields": [
            "phone",
            "mobile",
            "user.phone"
        ]
    }
}
```

**SQL:**

```sql
WHERE (
  phone ILIKE '%+7%'
  OR mobile ILIKE '%+7%'
  OR EXISTS (
    SELECT * FROM users
    WHERE orders.user_id = users.id
    AND users.phone ILIKE '%+7%'
  )
)
```

---

### 8. Поиск артикула (SKU) - числовой

**Запрос:**

```json
{
    "search": {
        "query": "12345",
        "fields": [
            "sku",
            "barcode",
            "article"
        ]
    }
}
```

**SQL:**

```sql
WHERE (
  (sku ILIKE '%12345%' OR sku = 12345)
  OR (barcode ILIKE '%12345%' OR barcode = 12345)
  OR (article ILIKE '%12345%' OR article = 12345)
)
```

**Найдет:**

- SKU: "12345" (точное совпадение)
- SKU: "ABC-12345-XYZ" (частичное совпадение)
- Barcode: "1234567890" (содержит "12345")

---

### 9. Комбинация с другими фильтрами

**Запрос:**

```json
{
    "search": {
        "query": "laptop",
        "fields": [
            "name",
            "description",
            "brand.name"
        ]
    },
    "where": {
        "status": "active",
        "price": {
            "operator": ">=",
            "value": 500
        }
    },
    "where_has": {
        "category": {
            "where": {
                "slug": "electronics"
            }
        }
    }
}
```

**SQL:**

```sql
WHERE (
  name ILIKE '%laptop%'
  OR description ILIKE '%laptop%'
  OR EXISTS (
    SELECT * FROM brands
    WHERE products.brand_id = brands.id
    AND brands.name ILIKE '%laptop%'
  )
)
AND status = 'active'
AND price >= 500
AND EXISTS (
  SELECT * FROM categories
  WHERE products.category_id = categories.id
  AND categories.slug = 'electronics'
)
```

---

### 10. Альтернативный синтаксис (через `value`)

**Запрос:**

```json
{
    "search": {
        "value": "search term",
        "fields": [
            "title",
            "content"
        ]
    }
}
```

Эквивалентно:

```json
{
    "search": {
        "query": "search term",
        "fields": [
            "title",
            "content"
        ]
    }
}
```

---

## Ограничения

### 1. Пустые значения игнорируются

```json
{
    "search": {
        "query": "",
        "fields": [
            "name"
        ]
    }
}
```

**Результат:** фильтр не применяется, возвращаются все записи.

### 2. Пустой массив полей игнорируется

```json
{
    "search": {
        "query": "laptop",
        "fields": []
    }
}
```

**Результат:** фильтр не применяется.

### 3. Несуществующие поля

Если указано несуществующее поле:

```json
{
    "search": {
        "query": "test",
        "fields": [
            "non_existent_field"
        ]
    }
}
```

**Результат:** SQL ошибка при выполнении запроса.

**Решение:** Валидация полей в Form Request:

```php
public function rules()
{
    return [
        'search.query' => 'required|string|max:255',
        'search.fields' => 'required|array|min:1',
        'search.fields.*' => 'string|in:name,description,sku,brand.name',
    ];
}
```

### 4. Несуществующие relations

При указании несуществующего relation:

```json
{
    "search": {
        "query": "test",
        "fields": [
            "non_existent_relation.field"
        ]
    }
}
```

**Результат:** Laravel выбросит исключение `RelationNotFoundException`.

---

## Сравнение с другими фильтрами

### Search vs Where с LIKE

**Search:**

```json
{
    "search": {
        "query": "laptop",
        "fields": [
            "name",
            "description"
        ]
    }
}
```

**Where с LIKE:**

```json
{
    "or_where": {
        "name": {
            "operator": "%like%",
            "value": "laptop",
            "type": "string"
        },
        "description": {
            "operator": "%like%",
            "value": "laptop",
            "type": "string"
        }
    }
}
```

**Разница:**

- `search` компактнее для множественных полей
- `search` автоматически добавляет точное совпадение для чисел
- `where` с `%like%` дает больше контроля (можно использовать разные операторы для каждого поля)

---

## Заключение

`SearchFilter` предоставляет:

✅ **Простоту** - компактный синтаксис для поиска по множеству полей  
✅ **Умный поиск** - автоматическое определение числовых значений  
✅ **Relations** - поддержка вложенных связей через точечную нотацию  
✅ **Регистронезависимость** - ILIKE для PostgreSQL  
✅ **Производительность** - использование нативных Laravel методов  
✅ **Гибкость** - комбинирование с другими фильтрами

**Используйте SearchFilter для:**

- Поиска по каталогу товаров
- Поиска пользователей по имени/email/телефону
- Поиска заказов по номеру/статусу
- Глобального поиска по сайту

**Не используйте SearchFilter для:**

- Точной фильтрации (используйте `where`)
- Диапазонов (используйте `where_between`)
- Сложных условий (используйте `where_has` с вложенными фильтрами)
