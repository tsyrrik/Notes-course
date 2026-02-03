## Вопрос: ParamConverter (автопреобразование параметров)
Версия: Symfony 8.0 (stable), LTS 7.4; PHP 8.4+ / 8.2+.

## Простой ответ
автомагия, которая превращает параметры маршрута (id/slug) в объекты (обычно Doctrine Entity) и подставляет в аргументы контроллера. В современных версиях чаще используют `#[MapEntity]` и value resolvers.

## Ответ
```php
#[Route('/users/{id}', name: 'user_show', requirements: ['id' => '\d+'])]
public function show(User $user): Response {
    // $user найден по id без ручного запроса
}
```

- Работает через DoctrineParamConverter для сущностей; можно писать свои конвертеры.
- Требования/ключ можно менять (`mapping`, `options`), или отключить `@ParamConverter` атрибутами.

## Примеры

1. `show(User $user)` подставляет сущность по `{id}`.
2. `#[MapEntity(mapping: ['slug' => 'slug'])]` для поиска по slug.
3. Отключение автоконвертации для кастомной логики.

## Доп. теория

1. ParamConverter исторически из SensioFrameworkExtraBundle; сейчас используется value resolver.
2. Для нестандартных параметров лучше писать собственный resolver.
