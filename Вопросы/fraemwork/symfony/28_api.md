## Вопрос: Как делать API в Symfony
Версия: Symfony 7.x (LTS 6.4), синтаксис и подходы 6.4+.

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
