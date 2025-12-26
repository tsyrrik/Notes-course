## Вопрос: Роутинг
Версия: Symfony 7.x (LTS 6.4), синтаксис и подходы 6.4+.

## Простой ответ
сопоставляет URL/метод с контроллером. Описывается в YAML/PHP/аннотациях/атрибутах.

## Ответ
```php
# config/routes.yaml
home:
  path: /
  controller: App\Controller\HomeController::index

// Атрибуты
#[Route('/users/{id}', name: 'user_show', methods: ['GET'])]
public function show(int $id) {}
```
