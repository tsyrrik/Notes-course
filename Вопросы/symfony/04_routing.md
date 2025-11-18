# Роутинг

Простыми словами: сопоставляет URL/метод с контроллером. Описывается в YAML/PHP/аннотациях/атрибутах.

```php
# config/routes.yaml
home:
  path: /
  controller: App\Controller\HomeController::index

// Атрибуты
#[Route('/users/{id}', name: 'user_show', methods: ['GET'])]
public function show(int $id) {}
```
