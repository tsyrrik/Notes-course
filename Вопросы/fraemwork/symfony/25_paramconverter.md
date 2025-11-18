# ParamConverter (автопреобразование параметров)

Простыми словами: автомагия, которая превращает параметры маршрута (id/slug) в объекты (обычно Doctrine Entity) и подставляет в аргументы контроллера.

```php
#[Route('/users/{id}', name: 'user_show', requirements: ['id' => '\d+'])]
public function show(User $user): Response {
    // $user найден по id без ручного запроса
}
```

- Работает через DoctrineParamConverter для сущностей; можно писать свои конвертеры.
- Требования/ключ можно менять (`mapping`, `options`), или отключить `@ParamConverter` атрибутами.
