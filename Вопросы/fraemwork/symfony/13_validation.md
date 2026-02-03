## Вопрос: Валидация
Версия: Symfony 8.0 (stable), LTS 7.4; PHP 8.4+ / 8.2+.

## Простой ответ
валидаторы через атрибуты/аннотации или YAML; проверяют DTO/сущности/Form данные.

## Ответ

### Компонент Validator

Компонент Symfony Validator реализует спецификацию JSR-303 (Bean Validation) из мира Java и предоставляет мощный, расширяемый механизм валидации данных. Валидация может применяться к DTO, сущностям, значениям форм и любым PHP-объектам. Правила описываются через PHP-атрибуты (рекомендуется), YAML, XML или PHP-код.

### Основные constraints (ограничения)

```php
use Symfony\Component\Validator\Constraints as Assert;

class RegisterDto
{
    #[Assert\NotBlank(message: 'Email обязателен')]
    #[Assert\Email(mode: 'strict')]
    public string $email;

    #[Assert\NotBlank]
    #[Assert\Length(min: 8, max: 128, minMessage: 'Пароль минимум {{ limit }} символов')]
    #[Assert\Regex(pattern: '/[A-Z]/', message: 'Нужна хотя бы одна заглавная буква')]
    #[Assert\NotCompromisedPassword]  // Проверяет через haveibeenpwned.com
    public string $password;

    #[Assert\NotBlank]
    #[Assert\Length(min: 2, max: 100)]
    public string $name;

    #[Assert\Range(min: 18, max: 150)]
    public int $age;

    #[Assert\Url]
    public ?string $website = null;

    #[Assert\Choice(choices: ['male', 'female', 'other'])]
    public ?string $gender = null;

    #[Assert\IsTrue(message: 'Необходимо принять условия')]
    public bool $termsAccepted;
}
```

### Полный список популярных constraints

| Constraint | Описание |
|-----------|----------|
| `NotBlank` | Не пустое значение |
| `NotNull` | Не null |
| `Email` | Валидный email |
| `Length` | Ограничение длины строки |
| `Range` | Числовой диапазон |
| `Regex` | Регулярное выражение |
| `Url` | Валидный URL |
| `Uuid` | Валидный UUID |
| `Choice` | Значение из списка |
| `Count` | Количество элементов коллекции |
| `Type` | Проверка типа |
| `Callback` | Произвольная логика |
| `UniqueEntity` | Уникальность в БД (Doctrine) |
| `Valid` | Каскадная валидация вложенного объекта |
| `File` / `Image` | Валидация файла/изображения |
| `PasswordStrength` | Оценка силы пароля (Symfony 6.3+) |
| `Compound` | Группировка нескольких constraints |

### Валидация в контроллере

```php
use Symfony\Component\Validator\Validator\ValidatorInterface;

class UserController extends AbstractController
{
    #[Route('/api/register', methods: ['POST'])]
    public function register(
        Request $request,
        ValidatorInterface $validator,
        EntityManagerInterface $em,
    ): JsonResponse {
        $dto = new RegisterDto();
        $dto->email = $request->getPayload()->getString('email');
        $dto->password = $request->getPayload()->getString('password');
        $dto->name = $request->getPayload()->getString('name');
        $dto->termsAccepted = $request->getPayload()->getBoolean('terms');

        $errors = $validator->validate($dto);
        if (count($errors) > 0) {
            $errorMessages = [];
            foreach ($errors as $error) {
                $errorMessages[$error->getPropertyPath()] = $error->getMessage();
            }
            return $this->json(['errors' => $errorMessages], 422);
        }

        // Создание пользователя...
        return $this->json(['status' => 'ok'], 201);
    }
}
```

### MapRequestPayload (Symfony 6.3+/7)

В Symfony 7 можно автоматически маппить тело запроса в DTO с валидацией через атрибут `#[MapRequestPayload]`:

```php
#[Route('/api/register', methods: ['POST'])]
public function register(
    #[MapRequestPayload] RegisterDto $dto,
    EntityManagerInterface $em,
): JsonResponse {
    // $dto уже провалидирован, при ошибках автоматически 422
    // ...
    return $this->json(['status' => 'ok'], 201);
}
```

### Группы валидации

Группы позволяют применять разные наборы правил в разных контекстах:

```php
class UserDto
{
    #[Assert\NotBlank(groups: ['registration', 'profile'])]
    #[Assert\Email(groups: ['registration', 'profile'])]
    public string $email;

    #[Assert\NotBlank(groups: ['registration'])]
    #[Assert\Length(min: 8, groups: ['registration'])]
    public ?string $password = null; // При обновлении профиля пароль не обязателен
}

// Валидация с группой
$errors = $validator->validate($dto, null, ['registration']);
// Или при обновлении:
$errors = $validator->validate($dto, null, ['profile']);
```

### Callback и Expression constraints

```php
class OrderDto
{
    public float $price;
    public float $discount;

    // Callback для произвольной логики
    #[Assert\Callback]
    public function validateDiscount(ExecutionContextInterface $context): void
    {
        if ($this->discount > $this->price) {
            $context->buildViolation('Скидка не может превышать цену')
                ->atPath('discount')
                ->addViolation();
        }
    }

    // Или Expression constraint
    #[Assert\Expression(
        expression: 'this.discount <= this.price',
        message: 'Скидка не может превышать цену'
    )]
    public float $discountAlt;
}
```

### Валидация коллекций и вложенных объектов

```php
class OrderDto
{
    #[Assert\NotBlank]
    public string $customerName;

    // Каскадная валидация: валидирует каждый OrderItemDto
    #[Assert\Valid]
    #[Assert\Count(min: 1, minMessage: 'Заказ должен содержать хотя бы один товар')]
    /** @var OrderItemDto[] */
    public array $items = [];
}

class OrderItemDto
{
    #[Assert\NotBlank]
    public string $productName;

    #[Assert\Positive]
    public int $quantity;

    #[Assert\PositiveOrZero]
    public float $price;
}
```

### Практические советы

- Используйте DTO для валидации входных данных, а не сущности напрямую -- это разделяет ответственность
- `#[Assert\Valid]` обязателен для каскадной валидации вложенных объектов -- без него дочерние объекты не проверяются
- Для уникальности в БД используйте `#[UniqueEntity]` на уровне класса сущности (не DTO)
- В Symfony 7 используйте `#[MapRequestPayload]` для автоматической десериализации и валидации в API-контроллерах
- Все сообщения об ошибках можно переводить через Translation компонент
- `#[Assert\Sequentially]` позволяет остановить валидацию при первой ошибке (например, не проверять формат email, если поле пустое)

## Примеры

1. DTO с `#[Assert\NotBlank]` и `#[Assert\Email]`.
2. `#[MapRequestPayload]` автоматом валидирует тело запроса.
3. Группы валидации `registration` и `profile`.

## Доп. теория

1. `#[Assert\Valid]` нужен для валидации вложенных объектов.
2. `UniqueEntity` применяется к сущностям, а не к DTO.
