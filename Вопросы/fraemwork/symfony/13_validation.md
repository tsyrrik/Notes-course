# Валидация

Простыми словами: валидаторы через атрибуты/аннотации или YAML; проверяют DTO/сущности/Form данные.

```php
use Symfony\Component\Validator\Constraints as Assert;

class RegisterDto {
    #[Assert\NotBlank]
    #[Assert\Email]
    public string $email;
    #[Assert\Length(min:8)]
    public string $password;
}
```

Проверка:
```php
$errors = $validator->validate($dto);
if (count($errors)) { /* вернуть ошибки */ }
```
