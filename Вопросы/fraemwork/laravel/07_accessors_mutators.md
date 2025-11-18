# Mutators и Accessors

Простыми словами: Accessor форматирует поле при чтении, Mutator меняет значение перед сохранением.
- Accessor: преобразует значение при чтении.
- Mutator: изменяет перед сохранением.
- С Laravel 9+ `Attribute` объединяет оба.
```php
class User extends Model {
    public function getNameAttribute($value) { return ucfirst($value); }
    public function setPasswordAttribute($value) { $this->attributes['password'] = bcrypt($value); }

    protected function name(): Attribute {
        return Attribute::make(
            get: fn($v) => ucfirst($v),
            set: fn($v) => strtolower($v),
        );
    }
}
```
