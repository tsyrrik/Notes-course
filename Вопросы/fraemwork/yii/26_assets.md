## Вопрос: Assets
Версия: Yii2 (ветка 2.0.x); Yii3 в разработке.

## Простой ответ
Обертка для CSS/JS, которая регистрируется и подключается в layout.

## Ответ
## Вопрос: Что такое AssetBundle?
Ответ: Обертка для CSS/JS, которая регистрируется и подключается в layout.

## Вопрос: Пример AssetBundle
Ответ:
```php
class AppAsset extends yii\\web\\AssetBundle {
    public $basePath = '@webroot';
    public $baseUrl = '@web';
    public $css = ['css/site.css'];
    public $js = ['js/site.js'];
}
```

## Вопрос: Как подключить?
Ответ: `AppAsset::register($this);` в layout.
