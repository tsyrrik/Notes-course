## Вопрос: Кастомные правила валидации
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Настраиваем собственные правила, если стандартных не хватает.

## Ответ

Когда встроенных правил валидации недостаточно, Laravel предоставляет несколько способов создания собственных правил: Closure-правила для быстрых одноразовых проверок, Rule-объекты для переиспользуемых правил и расширение Validator через макросы. В Laravel 11 интерфейс Rule-объектов упрощён — используется `Illuminate\Contracts\Validation\ValidationRule` с единственным методом `validate()`.

**Способ 1: Closure-правило (inline, для одноразовых проверок):**

```php
$request->validate([
    'username' => [
        'required',
        'string',
        function (string $attribute, mixed $value, Closure $fail) {
            if (str_contains($value, ' ')) {
                $fail("Поле {$attribute} не должно содержать пробелов.");
            }
        },
    ],
    'domain' => [
        'required',
        function (string $attribute, mixed $value, Closure $fail) {
            if (! checkdnsrr($value, 'A')) {
                $fail('Домен :attribute не существует.');
            }
        },
    ],
]);
```

**Способ 2: Rule Object (Laravel 11 — рекомендуемый подход):**

```bash
php artisan make:rule Uppercase
php artisan make:rule PhoneNumber
```

```php
// Новый синтаксис (Laravel 11) — интерфейс ValidationRule
namespace App\Rules;

use Closure;
use Illuminate\Contracts\Validation\ValidationRule;

class Uppercase implements ValidationRule
{
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        if (strtoupper($value) !== $value) {
            $fail('Поле :attribute должно быть в верхнем регистре.');
        }
    }
}

// Использование
$request->validate([
    'code' => ['required', 'string', new Uppercase],
]);
```

**Правило с параметрами конструктора:**

```php
class MaxWords implements ValidationRule
{
    public function __construct(
        private int $maxWords = 100
    ) {}

    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        $wordCount = str_word_count($value);
        if ($wordCount > $this->maxWords) {
            $fail("Поле :attribute содержит {$wordCount} слов, максимум — {$this->maxWords}.");
        }
    }
}

// Использование
$request->validate([
    'bio'     => ['required', new MaxWords(50)],
    'article' => ['required', new MaxWords(5000)],
]);
```

**Правило с доступом к другим полям (DataAwareRule) и валидатору (ValidatorAwareRule):**

```php
use Illuminate\Contracts\Validation\DataAwareRule;
use Illuminate\Contracts\Validation\ValidationRule;

class UniqueInArray implements ValidationRule, DataAwareRule
{
    protected array $data = [];

    public function setData(array $data): static
    {
        $this->data = $data;
        return $this;
    }

    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        // Проверить, что значение уникально среди всех элементов массива
        $items = $this->data['items'] ?? [];
        $values = array_column($items, 'product_id');

        if (count($values) !== count(array_unique($values))) {
            $fail('Товары в заказе не должны повторяться.');
        }
    }
}
```

**Правило для проверки с помощью внешнего сервиса (с DI):**

```php
class NotDisposableEmail implements ValidationRule
{
    public function __construct(
        private EmailVerificationService $emailService
    ) {}

    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        if ($this->emailService->isDisposable($value)) {
            $fail('Одноразовые email-адреса не допускаются.');
        }
    }
}

// Использование — Laravel автоматически разрешит DI
$request->validate([
    'email' => ['required', 'email', app(NotDisposableEmail::class)],
]);
```

**Способ 3: Расширение Validator (для глобально доступных правил):**

```php
// В AppServiceProvider::boot()
use Illuminate\Support\Facades\Validator;

Validator::extend('phone_ru', function ($attribute, $value, $parameters, $validator) {
    return preg_match('/^\+7\d{10}$/', $value);
});

// Кастомное сообщение (в resources/lang/ru/validation.php)
// 'phone_ru' => 'Поле :attribute должно быть в формате +7XXXXXXXXXX.',

// Использование как строковое правило
$request->validate([
    'phone' => 'required|phone_ru',
]);
```

**Implicit Rule — правило, которое работает даже для пустых значений:**

```php
class RequiredIfPublished implements ValidationRule
{
    // Реализовать ImplicitRule, чтобы правило срабатывало для пустых полей
    // В Laravel 11 достаточно пометить атрибутом
}
```

**Практический совет:** для одноразовых проверок используйте Closure, для переиспользуемых — Rule Object. Не дублируйте логику: если проверка нужна в нескольких Form Request, создайте Rule-класс. В Laravel 11 старый интерфейс `Rule` с методами `passes()` и `message()` по-прежнему работает, но рекомендуется новый `ValidationRule` с единым методом `validate()`. Правила можно комбинировать с встроенными: `['required', 'string', new Uppercase, new MaxWords(100)]`. Для сложных правил с зависимостями используйте конструктор с DI.

## Примеры

1. `Rule::unique('users', 'email')`.
2. Класс правила: `php artisan make:rule Uppercase`.
3. Closure‑правило: `function ($attr, $value, $fail) { ... }`.

## Доп. теория

1. Правила можно переиспользовать между формами.
2. Сложную бизнес‑логику лучше выносить в сервис.
