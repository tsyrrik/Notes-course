## Вопрос: Кастомные правила валидации
Версия: Laravel 11 (актуально; базовые принципы применимы к 10+).

## Простой ответ
Настраиваем собственные правила, если стандартных не хватает.

## Ответ
```php
// Closure
$request->validate([
    'name' => [
        'required',
        function ($attribute, $value, $fail) {
            if (strtoupper($value) !== $value) $fail('Имя должно быть в верхнем регистре');
        },
    ],
]);

// Rule Object
php artisan make:rule Uppercase
class Uppercase implements Rule {
    public function passes($attribute, $value) { return strtoupper($value) === $value; }
    public function message() { return ':attribute должен быть в верхнем регистре'; }
}
$request->validate(['name' => ['required', new Uppercase]]);
```
