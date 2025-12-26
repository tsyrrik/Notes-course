## Вопрос: REST API в Yii2
Версия: Yii2 (ветка 2.0.x); Yii3 в разработке.

## Простой ответ
Готовые REST-контроллеры, сериализатор и URL-правила.

## Ответ
## Вопрос: Что предоставляет Yii2 для REST?
Ответ: Готовые REST-контроллеры, сериализатор и URL-правила.

## Вопрос: Пример контроллера
Ответ:
```php
class PostController extends yii\\rest\\ActiveController {
    public $modelClass = Post::class;
}
```

## Вопрос: Пример UrlRule
Ответ:
```php
['class' => 'yii\\rest\\UrlRule', 'controller' => ['post']]
```

## Вопрос: Что еще важно?
Ответ:
- Форматы ответа через `contentNegotiator`.
- `fields()` и `extraFields()` для сериализации.
