# Формы

Простыми словами: Form Component строит формы, мапит данные на объекты, валидирует и рендерит через Twig.

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
