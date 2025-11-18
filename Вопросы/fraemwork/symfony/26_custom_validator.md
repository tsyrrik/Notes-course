# Кастомный валидатор

Простыми словами: когда стандартных правил не хватает, создаём свой Constraint + Validator и отмечаем поле атрибутом.

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
