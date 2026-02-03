## Вопрос: Enum (PHP 8.1)
Версия: PHP 8.4; где есть исторические детали — см. по тексту.

## Простой ответ
- Enum даёт фиксированный набор именованных значений с типовой безопасностью вместо «магических строк/чисел». Есть unit (без значения) и backed (со строкой/числом).

## Ответ

Перечисления (enums, PHP 8.1+) — специальный тип данных для представления фиксированного набора именованных значений. Заменяют «магические строки» и константы.

### Два вида enum

**Unit enum** — без привязки к скалярному значению:
```php
enum Suit {
    case Hearts;
    case Diamonds;
    case Clubs;
    case Spades;
}
```

**Backed enum** — каждый case имеет скалярное значение (`string` или `int`):
```php
enum Status: string {
    case Published = 'published';
    case Draft     = 'draft';
    case Archived  = 'archived';
}
```

### Основные методы

| Метод | Описание | Пример |
| --- | --- | --- |
| `cases()` | Все варианты (массив) | `Status::cases()` |
| `from($value)` | Backed → enum (бросает ValueError) | `Status::from('draft')` |
| `tryFrom($value)` | Backed → enum \| null | `Status::tryFrom('unknown')` → null |
| `->value` | Скалярное значение (backed) | `Status::Draft->value` → `'draft'` |
| `->name` | Имя case (string) | `Status::Draft->name` → `'Draft'` |

### Возможности enum

```php
enum Status: string {
    case Published = 'published';
    case Draft     = 'draft';
    case Archived  = 'archived';

    // Методы
    public function label(): string {
        return match($this) {
            self::Published => 'Опубликовано',
            self::Draft     => 'Черновик',
            self::Archived  => 'В архиве',
        };
    }

    // Статические методы
    public static function active(): array {
        return [self::Published, self::Draft];
    }
}

// Интерфейсы
interface HasColor { public function color(): string; }

enum Priority: int implements HasColor {
    case Low    = 1;
    case Medium = 2;
    case High   = 3;

    public function color(): string {
        return match($this) {
            self::Low    => 'green',
            self::Medium => 'yellow',
            self::High   => 'red',
        };
    }
}
```

### Ограничения

- **Нельзя** создать через `new` — singleton по nature
- **Нельзя** наследовать от enum или наследовать enum
- **Нет свойств** (только константы и методы)
- **Backed enum** поддерживает только `string` или `int`, не оба сразу
- **Нет bitmask** — для флагов нужна своя реализация

### Использование с БД / ORM

```php
// Laravel: $casts в модели
class Post extends Model {
    protected $casts = [
        'status' => Status::class, // автоматическое приведение
    ];
}

// Doctrine: type mapping
#[Column(type: 'string', enumType: Status::class)]
private Status $status;

// Ручное сохранение/восстановление
$row['status'] = $post->status->value;             // enum → string
$post->status = Status::from($row['status']);       // string → enum
```

## Примеры
```php
enum HttpCode: int {
    case OK = 200;
    case NotFound = 404;
}

function isOk(HttpCode $code): bool {
    return $code === HttpCode::OK;
}
```

## Доп. теория
- Enum — это объектный тип, сравнивается по идентичности (`===`).
- Backed enum поддерживает только `int` или `string`, не смешивая оба.
