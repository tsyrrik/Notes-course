# Property hooks (PHP 8.4)
Позволяют объявлять геттер/сеттер прямо в свойстве.
```php
public int $age {
    get => $this->ageInternal;
    set(int $v) => $this->ageInternal = max(0, $v);
}
```
Даёт лаконичный контроль доступа и валидацию без шаблонных методов.
