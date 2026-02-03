## Вопрос: Формы
Версия: Symfony 8.0 (stable), LTS 7.4; PHP 8.4+ / 8.2+.

## Простой ответ
Form Component строит формы, мапит данные на объекты, валидирует и рендерит через Twig.

## Ответ

### Что такое Form Component

Symfony Form Component -- это мощная подсистема для создания, обработки и валидации HTML-форм. Она обеспечивает полный цикл: определение структуры формы, рендеринг в HTML (через Twig), обработку данных из запроса, маппинг данных на объекты и валидацию. Компонент работает по принципу Data Transformer -- входные данные (строки из HTTP-запроса) трансформируются в нужные PHP-типы.

### Создание FormType

Каждая форма описывается как класс, наследующий `AbstractType`. Поля добавляются в методе `buildForm()`:

```php
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\EmailType;
use Symfony\Component\Form\Extension\Core\Type\PasswordType;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\Extension\Core\Type\ChoiceType;
use Symfony\Component\Form\Extension\Core\Type\SubmitType;
use Symfony\Component\Form\Extension\Core\Type\RepeatedType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;

class RegistrationFormType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder
            ->add('name', TextType::class, [
                'label' => 'Имя',
                'attr' => ['placeholder' => 'Введите имя'],
            ])
            ->add('email', EmailType::class, [
                'label' => 'Email',
            ])
            ->add('plainPassword', RepeatedType::class, [
                'type' => PasswordType::class,
                'first_options' => ['label' => 'Пароль'],
                'second_options' => ['label' => 'Повторите пароль'],
                'invalid_message' => 'Пароли не совпадают',
                'mapped' => false,  // Не маппится напрямую на свойство сущности
            ])
            ->add('role', ChoiceType::class, [
                'choices' => [
                    'Пользователь' => 'ROLE_USER',
                    'Модератор' => 'ROLE_MODERATOR',
                ],
                'label' => 'Роль',
            ])
            ->add('submit', SubmitType::class, ['label' => 'Зарегистрироваться']);
    }

    public function configureOptions(OptionsResolver $options): void
    {
        $options->setDefaults([
            'data_class' => User::class,          // Маппинг на сущность/DTO
            'validation_groups' => ['registration'], // Группы валидации
        ]);
    }
}
```

### Обработка формы в контроллере

```php
class RegistrationController extends AbstractController
{
    #[Route('/register', name: 'app_register')]
    public function register(
        Request $request,
        UserPasswordHasherInterface $hasher,
        EntityManagerInterface $em,
    ): Response {
        $user = new User();
        $form = $this->createForm(RegistrationFormType::class, $user);
        $form->handleRequest($request);

        if ($form->isSubmitted() && $form->isValid()) {
            // Обработка пароля (mapped: false, поэтому берём вручную)
            $plainPassword = $form->get('plainPassword')->getData();
            $user->setPassword($hasher->hashPassword($user, $plainPassword));

            $em->persist($user);
            $em->flush();

            $this->addFlash('success', 'Регистрация успешна!');
            return $this->redirectToRoute('app_login');
        }

        return $this->render('registration/register.html.twig', [
            'registrationForm' => $form,
        ]);
    }
}
```

### Рендеринг формы в Twig

```twig
{# Простой рендеринг -- вся форма сразу #}
{{ form(registrationForm) }}

{# Полный контроль над рендерингом #}
{{ form_start(registrationForm) }}
    {{ form_errors(registrationForm) }}

    <div class="form-group">
        {{ form_label(registrationForm.name) }}
        {{ form_widget(registrationForm.name, {'attr': {'class': 'form-control'}}) }}
        {{ form_errors(registrationForm.name) }}
    </div>

    <div class="form-group">
        {{ form_row(registrationForm.email) }}  {# label + widget + errors #}
    </div>

    {{ form_row(registrationForm.plainPassword.first) }}
    {{ form_row(registrationForm.plainPassword.second) }}

    {{ form_row(registrationForm.submit) }}
{{ form_end(registrationForm) }}
```

### Популярные типы полей

| Тип | Описание |
|-----|----------|
| `TextType` | Текстовое поле |
| `TextareaType` | Многострочное текстовое поле |
| `EmailType` | Email |
| `PasswordType` | Пароль |
| `IntegerType` | Целое число |
| `MoneyType` | Денежное значение |
| `DateType` / `DateTimeType` | Дата / Дата и время |
| `ChoiceType` | Выпадающий список / радиокнопки / чекбоксы |
| `EntityType` | Выбор Doctrine Entity (из БД) |
| `FileType` | Загрузка файла |
| `CollectionType` | Динамическая коллекция вложенных форм |
| `HiddenType` | Скрытое поле |
| `RepeatedType` | Два поля с проверкой совпадения |

### EntityType -- выбор из базы данных

```php
use Symfony\Bridge\Doctrine\Form\Type\EntityType;

$builder->add('category', EntityType::class, [
    'class' => Category::class,
    'choice_label' => 'name',
    'placeholder' => 'Выберите категорию',
    'query_builder' => function (CategoryRepository $repo) {
        return $repo->createQueryBuilder('c')
            ->where('c.active = true')
            ->orderBy('c.name', 'ASC');
    },
]);
```

### CollectionType -- динамические вложенные формы

```php
$builder->add('items', CollectionType::class, [
    'entry_type' => OrderItemType::class,
    'entry_options' => ['label' => false],
    'allow_add' => true,       // Разрешить добавление через JS
    'allow_delete' => true,    // Разрешить удаление
    'by_reference' => false,   // Вызывать addItem()/removeItem() на объекте
]);
```

### Data Transformers

Для преобразования данных между формой и моделью используются трансформеры:

```php
$builder->add('tags', TextType::class);
$builder->get('tags')->addModelTransformer(new CallbackTransformer(
    fn(?array $tags) => $tags ? implode(', ', $tags) : '',    // Из модели в форму
    fn(?string $str) => $str ? array_map('trim', explode(',', $str)) : [],  // Из формы в модель
));
```

### Form Events

Для динамической модификации формы на основе данных используются события:

```php
$builder->addEventListener(FormEvents::PRE_SET_DATA, function (FormEvent $event) {
    $user = $event->getData();
    $form = $event->getForm();

    // Показать поле "компания" только для бизнес-пользователей
    if ($user && $user->getType() === 'business') {
        $form->add('companyName', TextType::class, ['required' => true]);
    }
});
```

### Практические советы

- Используйте DTO вместо сущностей в `data_class` для сложных форм -- это разделяет слой представления и домен
- CSRF-защита включена по умолчанию -- не отключайте её без причины
- Для API-контроллеров формы обычно не нужны -- используйте `#[MapRequestPayload]` + Serializer + Validator
- В Symfony 7 при передаче формы в `render()` не нужно вызывать `->createView()` -- Twig делает это автоматически
- Используйте Bootstrap или Tailwind темы форм: `twig.form_themes: ['bootstrap_5_layout.html.twig']`

## Примеры

1. `RegistrationFormType` с `RepeatedType` для пароля.
2. `EntityType` для выбора категории из БД.
3. `CollectionType` для динамических позиций заказа.

## Доп. теория

1. CSRF включён по умолчанию и должен оставаться в веб‑формах.
2. Data Transformers полезны для строк ↔ коллекций.
