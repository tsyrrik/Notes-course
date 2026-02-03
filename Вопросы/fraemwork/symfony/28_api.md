## Вопрос: Как делать API в Symfony
Версия: Symfony 8.0 (stable), LTS 7.4; PHP 8.4+ / 8.2+.

## Простой ответ
можно руками через контроллеры (JsonResponse/serializer) или через API Platform. Базовый ручной пример:

## Ответ
```php
#[Route('/api/users', methods: ['GET'])]
public function index(UserRepository $repo): JsonResponse {
    $users = $repo->findAll();
    return $this->json($users, 200, [], ['groups' => 'user:read']);
}
```

- Для стабильных контрактов используйте нормализацию/группы `Serializer`, DTO, валидацию.
- Auth: JWT/Sanctum аналоги через Security + LexikJWT/Passport или сессии для SPA.
- Документация: NelmioApiDoc или OpenAPI/Swagger.
- Быстрый старт: API Platform добавляет ресурсы, CRUD, фильтры, пагинацию, Swagger UI.

## Примеры

1. `return $this->json($dto, 200, [], ['groups' => 'user:read']);`.
2. DTO + `#[MapRequestPayload]` для входящих данных.
3. API Platform ресурс `#[ApiResource]` для CRUD.

## Доп. теория

1. Для стабильных контрактов используйте Serializer groups и DTO.
2. Валидацию входа лучше держать отдельно от сущностей.
