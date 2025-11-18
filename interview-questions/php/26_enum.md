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
