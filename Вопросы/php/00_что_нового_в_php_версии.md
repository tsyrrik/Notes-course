## Вопрос: Что нового в PHP 7.0–8.4
Версия: PHP 8.4; где есть исторические детали — см. по тексту.

## Простой ответ
- Подборка фич по версиям: типы, строгий режим, match/атрибуты/JIT/enum/property hooks и пр. — чтобы быстро вспомнить, что появилось и когда.

## Ответ
Кратко по ключевым возможностям и изменениям в крупных релизах.

## PHP 7.0
- Исключения для фатальных ошибок (`Error` и производные), `TypeError`.
- Операторы `<=>`, `??`, `yield from`, новые правила для `list()`, `foreach`, побитовых опов.
- Анонимные классы, группировка `use`, улучшения замыканий.
- Скалярные типы параметров и возвращаемых значений, `declare(strict_types=1)`.
- Изменения: `func_get_args/arg`, `unserialize` с опцией, конструкторы PHP4 deprecated, новый синтаксис Unicode.
```php
declare(strict_types=1);

function cmp(int $a, int $b): int { return $a <=> $b; }
$user = $maybeUser ?? new class() {}; // оператор ?? и анонимный класс
```

## PHP 7.1
- `void`, `iterable`, nullable `?Type`, объединенные типы исключений `catch (A|B)`.
- Отрицательные смещения в строках, строковые ключи в `list()`.
- Callable -> closure, видимость констант класса.
```php
function logIt(?string $msg): void { echo $msg ?? '[empty]'; }
try { foo(); } catch (A|B $e) { /* один catch на несколько */ }
```

## PHP 7.2
- Загрузка расширений по имени, улучшения LSP-наследования.
- `object` как type hint, запятая в конце группированного namespace.
- `count()` предупреждает на непредсчитываемых значениях, `get_class(null)` запрещен.
- Libsodium в ядре, Argon2 в `password_hash()`, `JSON_INVALID_UTF8_*`.
```php
function asObject(object $o) { /* ... */ }
password_hash('secret', PASSWORD_ARGON2ID);
```

## PHP 7.3
- Гибкий heredoc/nowdoc, конечные запятые в аргументах.
- Ссылки в `list()`, `is_countable()`, `array_key_first/last()`.
- Deprecations: `image2wbmp()`, регистронезависимые константы, часть `FILTER_VALIDATE_URL` флагов.
```php
$text = <<<TXT
multi-line without indent issues
TXT;
foo(
    'a',
    'b',
); // trailing comma в вызове
```

## PHP 7.4
- Типизированные свойства, стрелочные функции `fn()`, оператор `??=`.
- Ковариантность/контравариантность, распаковка в массивах `...$arr`.
- Preloading, FFI для расширений.
```php
class User { public int $age; }
$nums = [1,2,3]; $sum = array_reduce($nums, fn($c,$v) => $c+$v, 0);
$config ??= loadDefaults();
```

## PHP 8.0
- Именованные аргументы, атрибуты `#[Attribute]`.
- Конструкторное promotion свойств, union types, `match`, nullsafe `?->`.
- JIT, улучшенное сравнение строк/чисел, строгая типизация встроенных функций.
```php
function box(int|string $v) { /* union */ }
$r = format(str: 'hi', repeat: 3); // именованные аргументы
$color = match($status) { 'ok' => 'green', default => 'gray' };
$middleName = $user?->profile?->middleName; // nullsafe
```

## PHP 8.1
- Intersection types, `never`, readonly-свойства, `enum`.
- Fibers, final-константы, first-class callable syntax.
- `array_is_list()`, AVIF/WebP улучшения, явная октальная нотация `0o777`.
```php
enum Status { case OK; case FAIL; }
class Repo { public readonly string $url; public function __construct(string $url){$this->url=$url;} }
$call = strlen(...); // first-class callable
```

## PHP 8.2
- Readonly-классы, DNF-типы, скалярные типы `true|false|null` отдельно.
- Константы в трейтах, новый `Random` extension (+SensitiveParameter).
- Deprecation динамических свойств, атрибут `AllowDynamicProperties`.
```php
readonly class Point { public function __construct(public int $x, public int $y) {} }
function check((A&B)|(C&D) $v) {}
$rng = new Random\Randomizer(); $rng->getInt(1, 10);
```

## PHP 8.3
- Типизированные константы класса, динамический доступ `MyClass::{expr}`.
- Атрибут `Override`, `json_validate()`, новые методы `Randomizer`.
- Улучшенные исключения DateTime, deprecation `get_class()` без аргумента и связанные функции.
```php
class C { public const int ID = 1; #[Override] public function toString() {} }
$valid = json_validate('{"a":1}');
$const = C::{strtoupper('id')}; // динамический доступ
```

## PHP 8.4
- Property Hooks, asymmetric visibility для свойств.
- Новый DOM API с HTML5, BcMath\Number (ООП arbitrary precision).
- Новые функции массивов: `array_find/_key/any/all()`.
- Lazy Objects, оптимизации `sprintf`, SHA-NI, создание объектов без скобок для чанинга.
- Обновленный цикл поддержки: 2 года фичи + 2 года безопасности.
```php
public string $name {
    get => $this->raw;
    set(string $v) => $this->raw = trim($v);
}
$firstEven = array_find([1,2,3,4], fn($v)=>$v%2===0); // 2
class Builder { public function value(): self { return $this; } }
$obj = Builder::value; // без скобок для чанинга
```
