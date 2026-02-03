## Вопрос: Property hooks (PHP 8.4)
Версия: PHP 8.4; где есть исторические детали — см. по тексту.

## Простой ответ
можно объявить `get/set` прямо на свойстве, без явных методов, с типами и валидацией — лаконичная замена шаблонным геттерам/сеттерам.

## Ответ

Property Hooks — одно из главных нововведений PHP 8.4. Позволяют определить `get` и `set` хуки прямо на свойстве класса, избавляя от шаблонных геттеров/сеттеров.

### Синтаксис

```php
class User {
    public string $name {
        get => strtoupper($this->name);
        set(string $value) => trim($value);
    }

    // Только get (виртуальное/вычисляемое свойство)
    public string $fullName {
        get => $this->firstName . ' ' . $this->lastName;
    }

    // Только set
    public string $password {
        set(string $value) {
            $this->password = password_hash($value, PASSWORD_BCRYPT);
        }
    }
}

$user = new User();
$user->name = '  alice  ';  // set hook: trim
echo $user->name;            // get hook: ALICE
```

### Краткая vs полная форма

```php
// Краткая форма (выражение)
public int $age {
    set(int $v) => max(0, $v);
}

// Полная форма (блок кода)
public int $age {
    set(int $v) {
        if ($v < 0) throw new \InvalidArgumentException('Age must be >= 0');
        $this->age = $v;
    }
}
```

### Виртуальные свойства

Если свойство имеет только `get` без `set`, или `get`/`set` не обращаются к `$this->propertyName`, оно становится **виртуальным** — не занимает память в объекте:

```php
class Circle {
    public function __construct(public float $radius) {}

    // Виртуальное свойство — не хранится, вычисляется
    public float $area {
        get => M_PI * $this->radius ** 2;
    }
}
```

### Asymmetric Visibility (PHP 8.4)

Совместно с property hooks часто используется асимметричная видимость:

```php
class Post {
    // Читать можно снаружи, записывать — только изнутри
    public private(set) string $title;

    public function __construct(string $title) {
        $this->title = $title;
    }
}

$post = new Post('Hello');
echo $post->title;     // OK
// $post->title = 'X'; // Error: cannot modify private(set) property
```

### Преимущества перед __get/__set

- **IDE-поддержка** — свойства видны в автокомплите, типизированы
- **Статический анализ** — PHPStan/Psalm работают корректно
- **Производительность** — быстрее магических методов
- **Явность** — контракт виден в определении класса

## Примеры
```php
class Price {
    public float $value {
        set(float $v) => $this->value = max(0, $v);
        get => round($this->value, 2);
    }
}
```

## Доп. теория
- Хуки не создают отдельные методы: это синтаксис доступа к свойству, который может валидировать/нормализовать данные.
- Виртуальные свойства (только `get`) не хранят значение в объекте и вычисляются при каждом чтении.
