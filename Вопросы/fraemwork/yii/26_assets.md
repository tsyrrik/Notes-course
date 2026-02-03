## Вопрос: Assets
Версия: Yii2 2.0.54; Yii3 релизнут (2025‑12‑31).

## Простой ответ
Обертка для CSS/JS, которая регистрируется и подключается в layout.

## Ответ

AssetBundle — обёртка для CSS/JS, регистрируется в layout и управляет зависимостями.

Пример AssetBundle:
```php
class AppAsset extends yii\\web\\AssetBundle {
    public $basePath = '@webroot';
    public $baseUrl = '@web';
    public $css = ['css/site.css'];
    public $js = ['js/site.js'];
}
```

Как подключить:
`AppAsset::register($this);` в layout.

## Примеры

1. Подключение `AppAsset` в `layouts/main.php`.
2. Указание `depends` для Bootstrap AssetBundle.
3. Добавление версии файлов через `appendTimestamp`.

## Доп. теория

1. AssetBundle умеет описывать зависимости и порядок загрузки.
2. Для продакшена полезна минимизация и объединение файлов.
