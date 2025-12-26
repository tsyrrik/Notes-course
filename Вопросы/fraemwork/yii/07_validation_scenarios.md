## Вопрос: Валидация и сценарии
Версия: Yii2 (ветка 2.0.x); Yii3 в разработке.

## Простой ответ
Правила задаются в `rules()` и применяются через `validate()`.

## Ответ
## Вопрос: Как описываются правила валидации?
Ответ: Правила задаются в `rules()` и применяются через `validate()`.

## Вопрос: Что такое сценарии?
Ответ: Сценарии позволяют применять разные правила для разных контекстов (create/update).

## Вопрос: Пример
Ответ:
```php
public function rules() {
    return [
        [['title', 'body'], 'required'],
        ['status', 'in', 'range' => [0,1]],
    ];
}

public function scenarios() {
    $scenarios = parent::scenarios();
    $scenarios['update'] = ['title'];
    return $scenarios;
}
```

## Вопрос: Как выглядит стандартный цикл формы?
Ответ: `load()` + `validate()` + `save()` (для AR).
