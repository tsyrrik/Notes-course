## Вопрос: Модели и ActiveRecord
Версия: Yii2 (ветка 2.0.x); Yii3 в разработке.

## Простой ответ
`yii\\base\\Model` — валидация и формы, `yii\\db\\ActiveRecord` — работа с БД по Active Record.

## Ответ
## Вопрос: Чем отличаются Model и ActiveRecord?
Ответ: `yii\\base\\Model` — валидация и формы, `yii\\db\\ActiveRecord` — работа с БД по Active Record.

## Вопрос: Пример ActiveRecord
Ответ:
```php
class Post extends yii\\db\\ActiveRecord {
    public static function tableName() { return 'post'; }
}

$post = Post::find()->where(['status' => 1])->all();
```

## Вопрос: Как работает массовое присваивание?
Ответ:
- Используй `rules()` и `safe`.
- `load()` заполняет только безопасные атрибуты.
