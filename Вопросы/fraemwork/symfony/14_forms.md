## Вопрос: Формы
Версия: Symfony 7.x (LTS 6.4), синтаксис и подходы 6.4+.

## Простой ответ
Form Component строит формы, мапит данные на объекты, валидирует и рендерит через Twig.

## Ответ
```php
class UserType extends AbstractType {
    public function buildForm(FormBuilderInterface $builder, array $options) {
        $builder
            ->add('email', EmailType::class)
            ->add('password', PasswordType::class)
            ->add('save', SubmitType::class);
    }
}

// В контроллере
$form = $this->createForm(UserType::class, $user);
$form->handleRequest($request);
if ($form->isSubmitted() && $form->isValid()) { /* ... */ }
```
