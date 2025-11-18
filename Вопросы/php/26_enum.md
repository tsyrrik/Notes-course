# Enum (PHP 8.1)
Перечисления задают фиксированный набор значений, заменяя «магические строки».
```php
enum Status: string {
    case PUBLISHED = 'published';
    case DRAFT = 'draft';
    case ARCHIVED = 'archived';
    public function color(): string { return match($this){
        self::PUBLISHED => 'green', self::DRAFT => 'blue', self::ARCHIVED => 'gray'}; }
}
```
- Unit enums или backed (`: string|int`).
- Методы: `cases()`, `from()/tryFrom()`.
- Создавать через `new` нельзя, неизменяемы.
- Можно добавлять методы, использовать в `match`, типизировать параметры/возвраты.

Плюсы:
- Безопасно вместо “магических строк/чисел”, строгая проверка типов.
- Удобные методы (`cases`, `from/tryFrom`), можно в `switch/match`, можно хранить метадату методами.
- Читаемость и рефакторинг (IDE/статанализ).

Минусы:
- Backed enums хранят строку/число, но не поддерживают множественные значения/флаги как bitmask (нужна своя реализация или сплит).
- Взаимодействие с ORM/DB требует явных маппингов (кастомные типы/касты).

Пример использования
```php
enum Status: string { case PUBLISHED='published'; case DRAFT='draft'; }

$status = Status::tryFrom('draft') ?? Status::DRAFT;
if ($status === Status::PUBLISHED) { /* ... */ }
```

## Простыми словами
- Enum даёт фиксированный набор именованных значений с типовой безопасностью вместо «магических строк/чисел». Есть unit (без значения) и backed (со строкой/числом).
