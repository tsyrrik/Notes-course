## Вопрос: REST API в Yii2
Версия: Yii2 2.0.54; Yii3 релизнут (2025‑12‑31).

## Простой ответ
Готовые REST-контроллеры, сериализатор и URL-правила.

## Ответ

Yii2 предоставляет REST‑контроллеры (`ActiveController`), сериализатор, контент‑нейготиэйшн и URL‑правила. Достаточно указать `modelClass`, и фреймворк создаст CRUD‑эндпоинты.

Пример контроллера:
```php
class PostController extends yii\\rest\\ActiveController {
    public $modelClass = Post::class;
}
```

Пример UrlRule:
```php
['class' => 'yii\\rest\\UrlRule', 'controller' => ['post']]
```

Что еще важно:
- Форматы ответа через `contentNegotiator`.
- `fields()` и `extraFields()` для сериализации.

## Примеры

1. `GET /posts` и `GET /posts/10` через `ActiveController`.
2. `fields()` скрывает `password_hash`, `extraFields()` добавляет `comments`.
3. `UrlRule` с `extraPatterns` для `POST publish`.

## Доп. теория

1. Для версии API используйте модули и префиксы (`/v1`, `/v2`).
2. Аутентификация для API обычно через `HttpBearerAuth` или `QueryParamAuth`.
