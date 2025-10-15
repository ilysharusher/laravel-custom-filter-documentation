# 🔍 Система фильтрации Laravel

Мощная и гибкая система фильтрации запросов с поддержкой вложенных условий, различных операторов сравнения и передачи
фильтров через URL или тело запроса.

---

## 📋 Оглавление

- [Основные возможности](#-основные-возможности)
- [Операторы сравнения](#-операторы-сравнения)
- [Базовые фильтры](#-базовые-фильтры)
- [Операторы для чисел](#-операторы-для-чисел)
- [Операторы для строк](#-операторы-для-строк)
- [Фильтрация по связям](#-фильтрация-по-связям)
- [Вложенные условия](#-вложенные-условия)
- [Передача фильтров в теле запроса](#-передача-фильтров-в-теле-запроса)
- [Пагинация и сортировка](#-пагинация-и-сортировка)
- [Специальные операторы](#-специальные-операторы)
- [Примеры реальных сценариев](#-примеры-реальных-сценариев)
- [Использование в коде](#-использование-в-коде)
- [Конфигурация](#️-конфигурация)
- [Архитектура](#-архитектура)
- [Преимущества](#-преимущества)

---

## ✨ Основные возможности

- **Множество операторов**: `=`, `!=`, `>`, `<`, `>=`, `<=`, `like`, `in`, `not in`, `is null`
- **Вложенные условия**: комбинирование `where`, `orWhere`, `whereNot`
- **Фильтрация по связям**: `whereHas`, `whereDoesntHave` с вложенными условиями
- **Регистронезависимый поиск**: автоматический `ILIKE` для PostgreSQL
- **POST-фильтры**: передача сложных фильтров через тело запроса
- **Пагинация**: встроенная поддержка с настройкой `itemsPerPage`

---

## 🎯 Операторы сравнения

```php
// Enum: App\Support\QueryBuilders\Enums\AdvancedFilterOperator

'='          // EQUALS - равно
'!='         // NOT_EQUALS - не равно
'>'          // GREATER_THAN - больше
'<'          // LESS_THAN - меньше
'>='         // GREATER_THAN_OR_EQUAL - больше или равно
'<='         // LESS_THAN_OR_EQUAL - меньше или равно
'like'       // LIKE - содержит (с ручными %)
'%like%'     // LIKE_ANYWHERE - содержит в любом месте
'%like'      // LIKE_ENDS_WITH - заканчивается на
'like%'      // LIKE_STARTS_WITH - начинается с
'not like'   // NOT_LIKE - не содержит
'in'         // IN - входит в список
'not in'     // NOT_IN - не входит в список
'is'         // IS - проверка на null/not_null
```

---

## 📌 Базовые фильтры

### 1. **WHERE** - простые условия

**Простое значение** (автоматически используется оператор `=`):

```
GET /api/users?where[status]=active
```

**Массив значений** (автоматически используется `IN`):

```
GET /api/users?where[status][]=active&where[status][]=pending
```

**С указанием оператора**:

```
GET /api/users?where[age][operator]=>=&where[age][value]=18
```

**Короткий синтаксис оператора**:

```
GET /api/users?where[age][op]=>=&where[age][value]=18
```

### 2. **OR WHERE** - условия с ИЛИ

**Поиск по нескольким полям**:

```
GET /api/users?orWhere[email]=test@example.com&orWhere[phone]=+380123456789
```

**Использование snake_case**:

```
GET /api/users?or_where[email]=test@example.com
```

### 3. **WHERE NOT** - отрицание

**Исключить значения**:

```
GET /api/users?whereNot[status]=banned
```

**С оператором** (инвертируется автоматически):

```
GET /api/users?whereNot[age][operator]=<&whereNot[age][value]=18
```

Выполнится как: `WHERE age >= 18`

**Альтернативный синтаксис**:

```
GET /api/users?where_not[role]=guest
```

---

## 🔢 Операторы для чисел

### Сравнение чисел

**Больше**:

```
GET /api/products?where[price][operator]=>&where[price][value]=100
```

**Меньше или равно**:

```
GET /api/products?where[stock][operator]=<=&where[stock][value]=10
```

**Диапазон** (комбинирование):

```
GET /api/products?where[price][operator]=>=&where[price][value]=50&whereNot[price][operator]=>&whereNot[price][value]=200
```

Результат: `WHERE price >= 50 AND price <= 200`

### IN / NOT IN для чисел

**Список ID**:

```
GET /api/users?where[id][]=1&where[id][]=5&where[id][]=10
```

**Явное указание оператора IN**:

```
GET /api/users?where[id][operator]=in&where[id][values][]=1&where[id][values][]=5
```

**Исключение значений**:

```
GET /api/users?where[status_id][operator]=not in&where[status_id][values][]=3&where[status_id][values][]=7
```

---

## 📝 Операторы для строк

### LIKE с автоматическими wildcards

**Содержит** (anywhere):

```
GET /api/users?where[name][operator]=%like%&where[name][value]=John
```

SQL: `WHERE name ILIKE '%John%'`

**Начинается с**:

```
GET /api/users?where[email][operator]=like%&where[email][value]=admin
```

SQL: `WHERE email ILIKE 'admin%'`

**Заканчивается на**:

```
GET /api/users?where[email][operator]=%like&where[email][value]=@example.com
```

SQL: `WHERE email ILIKE '%@example.com'`

**Ручные wildcards** (оператор like):

```
GET /api/users?where[phone][operator]=like&where[phone][value]=+380%
```

SQL: `WHERE phone ILIKE '+380%'`

### Регистронезависимый поиск

**С указанием типа string** (использует ILIKE вместо LIKE):

```
GET /api/users?where[name][operator]=%like%&where[name][value]=john&where[name][type]=string
```

### NOT LIKE

**Исключить строки**:

```
GET /api/users?where[email][operator]=not like&where[email][value]=%test%
```

SQL: `WHERE email NOT ILIKE '%test%'`

---

## 🔗 Фильтрация по связям

### WHERE HAS - наличие связи

**Простая проверка наличия**:

```
GET /api/users?whereHas[posts][where][status]=published
```

**Альтернативный синтаксис**:

```
GET /api/users?where_has[posts][where][status]=published
```

### WHERE DOESNT HAVE - отсутствие связи

**Пользователи без опубликованных постов**:

```
GET /api/users?whereDoesntHave[posts][where][status]=published
```

**Альтернативный синтаксис**:

```
GET /api/users?where_doesnt_have[posts][where][status]=published
```

---

## 🎭 Вложенные условия

### Многоуровневая вложенность связей

**Пользователи с постами, у которых есть комментарии от автора "John"**:

```
GET /api/users?whereHas[posts][whereHas][comments][where][author_name][operator]=%like%&whereHas[posts][whereHas][comments][where][author_name][value]=John
```

### Комбинирование условий в связях

**Комплексные условия**:

```
GET /api/orders?whereHas[items][where][quantity][operator]=>&whereHas[items][where][quantity][value]=5&whereHas[items][where][price][operator]=<&whereHas[items][where][price][value]=100&whereHas[items][whereNot][status]=cancelled
```

### WHERE и OR WHERE в связях

**Связь с OR условиями**:

```
GET /api/users?whereHas[posts][where][status]=published&whereHas[posts][orWhere][status]=draft
```

SQL: `WHERE EXISTS (SELECT * FROM posts WHERE (status = 'published' OR status = 'draft'))`

### WHERE NOT в связях

**Исключение значений в связях**:

```
GET /api/users?whereHas[orders][whereNot][status]=cancelled&whereHas[orders][where][total][operator]=>&whereHas[orders][where][total][value]=1000
```

---

## 📮 Передача фильтров в теле запроса

Система **полностью поддерживает** передачу фильтров в теле запроса для любых сложных запросов.

### 🔧 Поддерживаемые HTTP методы:

- ✅ **POST** - основной метод для длинных фильтров
- ✅ **PUT** / **PATCH** - можно использовать для обновления с фильтрацией
- ✅ **GET** - только через URL параметры
- ✅ **DELETE** - можно передавать фильтры в теле

### 📦 Поддерживаемые форматы:

#### 1. **JSON** (рекомендуется)

```http
POST /api/users
Content-Type: application/json

{
  "where": {
    "status": "active",
    "age": {
      "operator": ">=",
      "value": 18
    }
  },
  "page": 1,
  "itemsPerPage": 50
}
```

#### 2. **Form Data** (multipart/form-data)

```text
POST /api/users
Content-Type: multipart/form-data

where[status]=active
where[age][operator]=>=
where[age][value]=18
page=1
itemsPerPage=50
```

#### 3. **URL Encoded** (application/x-www-form-urlencoded)

```text
POST /api/users
Content-Type: application/x-www-form-urlencoded

where[status]=active&where[age][operator]=>=&where[age][value]=18&page=1&itemsPerPage=50
```

### 🎯 Важно:

- Laravel автоматически парсит **все форматы** через `request()->input()`
- Можно **комбинировать** URL параметры и тело запроса (они мёрджатся)
- Приоритет имеют данные из **тела запроса** при совпадении ключей

---

### Примеры использования

Для длинных и сложных фильтров используйте **POST запросы с JSON**:

### Базовый пример

```http
POST /api/users
Content-Type: application/json

{
  "where": {
    "status": "active",
    "age": {
      "operator": ">=",
      "value": 18
    }
  },
  "whereNot": {
    "role": "guest"
  },
  "page": 1,
  "itemsPerPage": 50
}
```

### Сложный фильтр со связями

```http
POST /api/orders
Content-Type: application/json

{
  "where": {
    "created_at": {
      "operator": ">=",
      "value": "2025-01-01"
    },
    "status": ["pending", "processing", "completed"]
  },
  "whereHas": {
    "customer": {
      "where": {
        "country": "US",
        "email": {
          "operator": "%like%",
          "value": "@example.com"
        }
      }
    },
    "items": {
      "where": {
        "price": {
          "operator": ">",
          "value": 50
        }
      },
      "whereNot": {
        "status": "cancelled"
      }
    }
  },
  "whereDoesntHave": {
    "refunds": {}
  },
  "sortBy": [
    {
      "key": "created_at",
      "order": "desc"
    }
  ],
  "itemsPerPage": 25
}
```

### Очень сложный вложенный фильтр

```http
POST /api/projects
Content-Type: application/json

{
  "where": {
    "status": {
      "operator": "in",
      "values": ["active", "planning", "in_progress"]
    },
    "budget": {
      "operator": ">=",
      "value": 10000
    }
  },
  "whereHas": {
    "tasks": {
      "where": {
        "priority": "high"
      },
      "whereHas": {
        "assignee": {
          "where": {
            "department": "development",
            "experience_level": {
              "operator": ">=",
              "value": 3
            }
          },
          "whereDoesntHave": {
            "vacations": {
              "where": {
                "start_date": {
                  "operator": "<=",
                  "value": "2025-12-31"
                },
                "end_date": {
                  "operator": ">=",
                  "value": "2025-01-01"
                }
              }
            }
          }
        }
      },
      "whereNot": {
        "status": ["cancelled", "completed"]
      }
    }
  },
  "orWhere": {
    "manager_id": 5,
    "deputy_manager_id": 5
  },
  "sortBy": [
    {
      "key": "priority",
      "order": "desc"
    },
    {
      "key": "created_at",
      "order": "asc"
    }
  ],
  "page": 1,
  "itemsPerPage": 20
}
```

---

## 📊 Пагинация и сортировка

### Базовая пагинация

```
GET /api/users?page=2&itemsPerPage=50
```

### Множественная сортировка

```
GET /api/users?sortBy[0][key]=created_at&sortBy[0][order]=desc&sortBy[1][key]=name&sortBy[1][order]=asc
```

### В теле запроса

```json
{
    "sortBy": [
        {
            "key": "priority",
            "order": "desc"
        },
        {
            "key": "name",
            "order": "asc"
        }
    ],
    "page": 1,
    "itemsPerPage": 100
}
```

### Выбор полей

**URL параметры**:

```
GET /api/users?fields[]=id&fields[]=name&fields[]=email
```

**В теле запроса**:

```json
{
    "fields": [
        "id",
        "name",
        "email"
    ],
    "where": {
        "status": "active"
    }
}
```

---

## 🔍 Специальные операторы

### IS NULL / IS NOT NULL

**Проверка на NULL**:

```
GET /api/users?where[deleted_at][operator]=is&where[deleted_at][value]=null
```

**Проверка на NOT NULL**:

```
GET /api/users?where[email_verified_at][operator]=is&where[email_verified_at][value]=not_null
```

### Поиск по кастомным полям

Поиск с автоматическим ILIKE по нескольким полям (параметр `filter`):

```
GET /api/users?filter[name]=John&filter[email]=example.com
```

Выполняется как: `WHERE (name ILIKE '%John%' OR email ILIKE '%example.com%')`

---

## 💡 Примеры реальных сценариев

### 1. E-commerce: Фильтр товаров

```http
POST /api/products
Content-Type: application/json

{
  "where": {
    "status": "active",
    "price": {
      "operator": ">=",
      "value": 10
    }
  },
  "whereNot": {
    "price": {
      "operator": ">",
      "value": 1000
    }
  },
  "whereHas": {
    "category": {
      "where": {
        "slug": ["electronics", "computers"]
      }
    },
    "reviews": {
      "where": {
        "rating": {
          "operator": ">=",
          "value": 4
        }
      }
    }
  },
  "whereDoesntHave": {
    "outOfStockNotifications": {}
  },
  "sortBy": [
    {
      "key": "popularity",
      "order": "desc"
    }
  ]
}
```

### 2. CRM: Поиск клиентов

```http
POST /api/customers
Content-Type: application/json

{
  "where": {
    "created_at": {
      "operator": ">=",
      "value": "2025-01-01"
    }
  },
  "orWhere": {
    "email": {
      "operator": "%like%",
      "value": "@corporate.com"
    },
    "phone": {
      "operator": "like%",
      "value": "+380"
    }
  },
  "whereHas": {
    "orders": {
      "where": {
        "total": {
          "operator": ">",
          "value": 5000
        },
        "status": ["completed", "shipped"]
      }
    }
  },
  "whereDoesntHave": {
    "complaints": {
      "where": {
        "resolved": false
      }
    }
  }
}
```

### 3. HR: Подбор сотрудников

```http
POST /api/employees
Content-Type: application/json

{
  "where": {
    "employment_status": "active",
    "experience_years": {
      "operator": ">=",
      "value": 2
    }
  },
  "whereHas": {
    "skills": {
      "where": {
        "name": ["PHP", "Laravel", "PostgreSQL"]
      }
    },
    "department": {
      "where": {
        "location": "Remote"
      },
      "whereNot": {
        "name": "Support"
      }
    }
  },
  "whereDoesntHave": {
    "vacations": {
      "where": {
        "end_date": {
          "operator": ">",
          "value": "2025-10-15"
        }
      }
    }
  }
}
```

---

## 🚀 Использование в коде

### В контроллере

```php
use App\Models\User;

class UserController extends Controller
{
    public function index()
    {
        // Автоматически применяются все фильтры из request()
        return User::query()->paginateFiltered('users.created_at');
    }
    
    public function search()
    {
        // С кастомной сортировкой по умолчанию
        $users = User::query()
            ->filter('users.name')
            ->paginate(25);
            
        return response()->json($users);
    }
}
```

### В модели

```php
use App\Support\QueryBuilders\CustomQueryBuilder;
use Illuminate\Database\Eloquent\Model;

class Product extends Model
{
    public static string $DEFAULT_SORT_DIRECTION = 'asc';

    public function newEloquentBuilder($query): CustomQueryBuilder
    {
        return new CustomQueryBuilder($query);
    }
}
```

### Добавление кастомных RAW условий

```php
Product::query()
    ->addRawFilterConditions([
        'products.sku LIKE ?' => 'search',
        'products.barcode LIKE ?' => 'search'
    ])
    ->paginateFiltered();
```

---

## ⚙️ Конфигурация

### Константы по умолчанию

```php
CustomQueryBuilder::DEFAULT_SORT_DIRECTION = 'desc';
CustomQueryBuilder::DEFAULT_PER_PAGE = 20;
```

### В модели

```php
class User extends Model
{
    // Переопределить направление сортировки
    public static string $DEFAULT_SORT_DIRECTION = 'asc';
    
    // Переопределить количество элементов на странице
    protected $perPage = 50;
}
```

---

## 📘 Архитектура

### Основные компоненты

1. **CustomQueryBuilder** - главный класс, расширяющий `Illuminate\Database\Eloquent\Builder`
2. **Фильтры**:
    - `WhereFilter` - обработка WHERE условий
    - `OrWhereFilter` - обработка OR WHERE условий
    - `WhereNotFilter` - обработка WHERE NOT условий
    - `WhereHasFilter` - фильтрация по связям
    - `WhereDoesntHaveFilter` - отсутствие связей
    - `CustomFieldFilter` - поиск по нескольким полям

3. **Support классы**:
    - `ConditionFactory` - создание условий из различных форматов
    - `OperatorExecutor` - применение операторов к запросу
    - `ConditionPayload` - DTO для передачи условий
    - `AbstractRelationFilter` - базовый класс для фильтров связей

4. **Enum**:
    - `AdvancedFilterOperator` - список поддерживаемых операторов

---

## 🎯 Преимущества

✅ **Единый синтаксис** - одинаковый формат для URL и POST запросов  
✅ **Типобезопасность** - использование Enum для операторов  
✅ **Расширяемость** - легко добавить новые операторы и фильтры  
✅ **Производительность** - ленивое применение фильтров  
✅ **Читаемость** - понятный DSL для фильтрации  
✅ **Вложенность** - неограниченная глубина вложенных условий

---

**Версия документации**: 1.0  
**Дата**: 2025-10-15
