## Вопрос: Формы и ActiveForm
Версия: Yii2 (ветка 2.0.x); Yii3 в разработке.

## Простой ответ
Виджет для генерации форм с привязкой к модели и валидацией.

## Ответ
## Вопрос: Что такое ActiveForm?
Ответ: Виджет для генерации форм с привязкой к модели и валидацией.

## Вопрос: Пример формы
Ответ:
```php
use yii\\widgets\\ActiveForm;

$form = ActiveForm::begin();
echo $form->field($model, 'title');
echo $form->field($model, 'body')->textarea();
ActiveForm::end();
```

## Вопрос: Как работает валидация?
Ответ: `load()` + `validate()` на модели, ошибки отображаются в полях.
