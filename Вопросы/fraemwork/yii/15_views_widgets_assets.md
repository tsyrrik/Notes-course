## Вопрос: Views, виджеты и ассеты
Версия: Yii2 2.0.54; Yii3 релизнут (2025‑12‑31).

## Простой ответ
View — шаблоны, виджеты — переиспользуемые UI-части, ассеты — управление CSS/JS.

## Ответ

View — это PHP‑шаблон; виджеты — переиспользуемые компоненты UI; ассеты — управление CSS/JS через `AssetBundle`.

Пример виджета:
```php
echo yii\\widgets\\ListView::widget([
    'dataProvider' => $dataProvider,
    'itemView' => '_item',
]);
```

Как подключаются ассеты:
- `AssetBundle` подключает CSS/JS.
- `AppAsset` регистрируется в layout.

## Примеры

1. `<?= Html::encode($model->title) ?>` в view для защиты от XSS.
2. `ListView` для списка постов с пагинацией.
3. `AppAsset::register($this);` в `views/layouts/main.php`.

## Доп. теория

1. Виджеты могут иметь собственные view‑файлы и ассеты.
2. Ассеты можно объединять/минифицировать через asset‑manager.
