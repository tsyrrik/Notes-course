## Вопрос: ParamConverter (автопреобразование параметров)
Версия: Symfony 7.x (LTS 6.4), синтаксис и подходы 6.4+.

## Простой ответ
автомагия, которая превращает параметры маршрута (id/slug) в объекты (обычно Doctrine Entity) и подставляет в аргументы контроллера.

## Ответ
```php
#[Route('/users/{id}', name: 'user_show', requirements: ['id' => '\d+'])]
public function show(User $user): Response {
    // $user найден по id без ручного запроса
}
```

- Работает через DoctrineParamConverter для сущностей; можно писать свои конвертеры.
- Требования/ключ можно менять (`mapping`, `options`), или отключить `@ParamConverter` атрибутами.
