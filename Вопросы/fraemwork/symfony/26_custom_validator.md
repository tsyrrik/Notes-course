## Вопрос: Кастомный валидатор
Версия: Symfony 8.0 (stable), LTS 7.4; PHP 8.4+ / 8.2+.

## Простой ответ
когда стандартных правил не хватает, создаём свой Constraint + Validator и отмечаем поле атрибутом.

## Ответ
```php
# Constraint
#[Attribute]
class UniqueEmail extends Constraint {
    public string $message = 'Email "{{ value }}" уже используется.';
}

# Validator
class UniqueEmailValidator extends ConstraintValidator {
    public function __construct(private UserRepository $repo) {}
    public function validate($value, Constraint $constraint): void {
        if (!$value) return;
        if ($this->repo->findOneBy(['email'=>$value])) {
            $this->context->buildViolation($constraint->message)->addViolation();
        }
    }
}
```

Применение:
```php
class UserDto {
    #[UniqueEmail]
    public string $email;
}
```

## Примеры

1. `UniqueEmail` проверяет уникальность в БД.
2. Валидатор `PhoneNumber` с regex и кастомным сообщением.
3. Инъекция репозитория в validator через autowiring.

## Доп. теория

1. Constraint — это метаданные, Validator — реализация проверки.
2. Для сложных правил лучше писать отдельный сервис и вызывать его из валидатора.
