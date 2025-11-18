# Как делать API в Symfony

Простыми словами: можно руками через контроллеры (JsonResponse/serializer) или через API Platform. Базовый ручной пример:

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
