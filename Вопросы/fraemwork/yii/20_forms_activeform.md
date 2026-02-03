## Вопрос: Формы и ActiveForm
Версия: Yii2 2.0.54; Yii3 релизнут (2025‑12‑31).

## Простой ответ
Виджет для генерации форм с привязкой к модели и валидацией.

## Ответ

ActiveForm — виджет для генерации форм с привязкой к модели и валидацией.

Пример формы:
```php
use yii\\widgets\\ActiveForm;

$form = ActiveForm::begin();
echo $form->field($model, 'title');
echo $form->field($model, 'body')->textarea();
ActiveForm::end();
```

Как работает валидация:
`load()` + `validate()` на модели, ошибки отображаются в полях.

## Примеры

1. Форма с `ActiveForm` и `Html::submitButton()`.
2. AJAX‑валидация через `enableAjaxValidation`.
3. Горизонтальная форма с `fieldConfig`.

## Доп. теория

1. ActiveForm опирается на `Model::rules()` и сценарии.
2. При AJAX‑валидации нужно обрабатывать ответы JSON.
